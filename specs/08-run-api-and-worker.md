# Spec 08 · Run API and worker

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** RUN · **ADRs:** ADR-002, ADR-003, ADR-013, ADR-015, ADR-016, ADR-017, ADR-019 · **Ground rules:** GR-04, GR-08, GR-09, GR-24, GR-27 to GR-30

The Run API is how an engagement team submits a ledger, starts runs and collects working papers. Each run is a batch job that one worker carries through its stages: prepare, score (spec 03), build cases (spec 04), run the agents (specs 05 and 06), and produce the working paper (spec 07). The run store keeps every input and output, so a run survives a restart, a lost worker or a regional failover.

## 1. Purpose

The brief needs up to 12 engagements running at once, each in under 4 hours, with results that survive a service restart and a ranking that repeats exactly (brief §§4–5, scope §2). This spec fixes the API, the job queue, the worker's stages and the run record, so that every run is versioned, resumable and traceable.

## 2. Scope

**In scope:**
- The API's resources and authentication.
- Uploads, the job table and the workers.
- Run stages, resuming, the run manifest, the decision hash, the deadline fallback and cancelling.

**Out of scope:**
- What each stage computes (specs 01 to 07).
- The cloud setup, deployment, telemetry and failover steps (spec 09).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| RUN-01 | Tickmark shall offer the HTTPS endpoints in §4.1, with JSON bodies and the error format in §4.1. | ADR-002 |
| RUN-02 | Tickmark shall authenticate every request except `/health` with a bearer token, and check on every request that the token's user may access the engagement in the path. | GR-24, ADR-013 |
| RUN-03 | If a token is valid but lacks access to the engagement, then Tickmark shall answer 404, as if the engagement did not exist. | ADR-013 |
| RUN-04 | Tickmark shall store tokens only as SHA-256 hashes, issue and revoke them with an admin command, and record each token's user and engagements. | GR-26 |
| RUN-05 | Tickmark shall receive submission files through upload links into the run store, valid for 15 minutes, and validate them as a queued job once the client marks the upload complete. | ADR-016 |
| RUN-06 | When a run is requested with an `Idempotency-Key` already used for that engagement, Tickmark shall return the existing run instead of starting another. | Scope §5 |
| RUN-07 | Tickmark shall build the run manifest (§4.3) when a run is created, and shall refuse the run if any item is missing. | GR-08 |
| RUN-08 | Tickmark shall queue every validation and run as a job in PostgreSQL. Workers shall claim jobs with `SELECT … FOR UPDATE SKIP LOCKED`, oldest first. | ADR-016 |
| RUN-09 | Tickmark shall start one worker task per claimed job, with at most 12 run workers at once. Further jobs wait in the queue. | ADR-016, brief §4 |
| RUN-10 | Tickmark's workers shall record a heartbeat every 30 seconds. If a job's heartbeat stops for 2 minutes, then Tickmark shall queue it again and start a replacement worker, up to 3 attempts. | GR-29 |
| RUN-11 | Tickmark shall run each job's stages in the order of §4.2 and write each stage's output under its run and stage keys, so that a restarted worker skips finished stages and resumes the agent stage from LangGraph's checkpoints. | GR-29, ADR-017 |
| RUN-12 | Tickmark shall pin each run to the worker image digest it started with, so that a deploy never changes a run in flight. | GR-30, ADR-019 |
| RUN-13 | Tickmark shall compute the run's decision hash from the scoring hash (spec 03) and the case hash (spec 04), as in §4.4, and record it. | ADR-003, GR-04 |
| RUN-14 | If the agent stage is still running 3 hours 15 minutes after the run started, then Tickmark shall finish the remaining cases with templates, so that the run ends within 4 hours. | GR-27 |
| RUN-15 | When a run is cancelled, Tickmark shall stop it at the next node boundary, keep what it has stored, and mark it cancelled. | ADR-002 |
| RUN-16 | Tickmark shall keep a run record (§4.5) for every run, readable through the API while the run is in progress and after it ends. | Scope §2 |
| RUN-17 | Tickmark shall keep every run and its outputs, and shall never overwrite them. A re-run is a new run. | GR-09 |
| RUN-18 | Tickmark shall enforce engagement scoping in PostgreSQL with row-level security on the engagement ID, and in the run store with one prefix per engagement. | GR-24 |
| RUN-19 | Tickmark shall answer status requests within 300 ms at the 95th percentile and 1 second at the 99th, with 12 runs in progress. | Scope §4 |
| RUN-20 | Tickmark shall resume a run in Hyderabad, after a failover, from the same stored state, and reach the same decision hash. | ADR-015, GR-04 |

## 4. Data shapes

### 4.1 Endpoints

All paths start with `/v1`. `{e}` is the engagement ID, `{s}` a submission ID and `{r}` a run ID.

| Method and path | Does |
|---|---|
| `POST /engagements/{e}/submissions` | Creates a submission, and returns an upload link for each of the four files |
| `POST /engagements/{e}/submissions/{s}/complete` | Marks the upload complete and queues validation |
| `GET /engagements/{e}/submissions/{s}` | Status (`uploading`, `validating`, `accepted` or `refused`) and the snapshot hash |
| `GET /engagements/{e}/submissions/{s}/report/{part}` | The validation report: `summary.json`, `rejected_rows.csv` or `warnings.csv` |
| `POST /engagements/{e}/runs` | Starts a run from a submission ID and run settings, with an `Idempotency-Key` header |
| `GET /engagements/{e}/runs` | Lists the engagement's runs, newest first |
| `GET /engagements/{e}/runs/{r}` | The run record (§4.5) |
| `POST /engagements/{e}/runs/{r}/cancel` | Cancels the run |
| `GET /engagements/{e}/runs/{r}/working-paper` | A download link for the PDF |
| `GET /engagements/{e}/runs/{r}/appendices/{name}` | A download link for an appendix |
| `GET /health` | Liveness, without authentication |

Errors return `{"code": "…", "message": "…", "details": […]}` with a fitting HTTP status: 400 invalid input, 401 no valid token, 404 unknown or not permitted, 409 conflicting state, 413 too large, 429 too many requests.

Engagements and tokens are created by admin commands, not through the API.

### 4.2 Stages

| Stage | Does | Output |
|---|---|---|
| `prepare` | Loads the snapshot, checks its hash, and validates the run settings and the manifest | The pinned manifest |
| `score` | Runs spec 03 | The scoring record and scoring hash |
| `cases` | Runs spec 04 | The case files and case hash, then the decision hash |
| `agents` | Runs specs 05 and 06, case by case | Findings, drafts, outcomes and model call records |
| `paper` | Runs spec 07 | The working paper and appendices |

A validation job has one stage, `validate`, which runs spec 01. Statuses are `queued`, `running`, `succeeded`, `failed` and `cancelled`.

### 4.3 Run manifest

The manifest is canonical JSON (RFC 8785), and its SHA-256 is the manifest hash. It holds:
- the snapshot hash, the submission ID and the settings hash;
- the format, rule bundle, catalogue, agent profile, prompt, verifier and template versions;
- each role's model ID and inference profile;
- the cited standard editions (ADR-022);
- the worker image digest.

The evaluation records each generated ledger's seed beside its run ID (spec 10).

### 4.4 Decision hash

The decision hash is the SHA-256 of the text `tickmark-decision-v1`, a line break, the scoring hash, a line break and the case hash. Timestamps, costs, model call records and agent wording are left out, so the decision hash depends only on the snapshot and the run settings (scope §5).

### 4.5 Run record

| Field | Meaning |
|---|---|
| `run_id`, `engagement_id`, `submission_id`, `number` | Which run this is, and its position among the engagement's runs |
| `status`, `stage`, `progress` | Where it is, and the cases finished out of the total |
| `started_at`, `finished_at`, `deadline_fallback_at` | Times, when known |
| `counts` | Entries scored, flagged and selected; cases; entries sent to the agents; model calls made and replayed |
| `cost` | Worst-case model cost, measured model cost, tokens |
| `outcomes` | Justification outcomes by reason (spec 06) |
| `hashes` | Manifest, snapshot, settings, scoring, case and decision hashes |
| `errors` | Any failures, with codes |

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| RUN-T01 | A generated 400,000-entry ledger | It is uploaded, validated, run and its paper downloaded through the API | Every endpoint answers as in §4.1, and the run succeeds within 4 hours. *Ordinary* | RUN-01, RUN-05, RUN-16 |
| RUN-T02 | Requests with no token, a revoked token, and a valid token for another engagement | They call a run endpoint | They get 401, 401 and 404. *Hostile: unauthorised engagement access* | RUN-02, RUN-03 |
| RUN-T03 | The token table | It is read directly | It holds only hashes, each with its user and engagements. | RUN-04 |
| RUN-T04 | An upload link used after 15 minutes | The upload is tried | It is refused. | RUN-05 |
| RUN-T05 | The same run request sent twice with one `Idempotency-Key` | Both arrive | One run exists, and both get its ID. *Hostile: repeated submissions* | RUN-06 |
| RUN-T06 | Run settings that omit `review_budget`, and an agent profile file that is missing | A run is requested | It is refused, and the error names the missing item. | RUN-07 |
| RUN-T07 | Two workers claiming jobs at the same moment | Both run the claim | Each job is claimed once. | RUN-08 |
| RUN-T08 | 14 runs requested at once | They start | 12 run at once and 2 wait, then start as others finish. | RUN-09 |
| RUN-T09 | A worker killed during the agent stage | Its heartbeat stops | The job is queued again within 3 minutes, a new worker resumes from the checkpoints without repeating finished cases, and the decision hash matches an uninterrupted run. *Recovery: worker termination and retry* | RUN-10, RUN-11 |
| RUN-T10 | A deploy of a new image while a run is in its agent stage | The deploy finishes | The run finishes on its original image, and the next run uses the new one. *Recovery: rollback and deploy* | RUN-12 |
| RUN-T11 | The same snapshot and settings run twice | Both finish | Their decision hashes match, while their timestamps and costs differ. | RUN-13 |
| RUN-T12 | A run whose agent stage is slowed to pass 3 hours 15 minutes | The deadline passes | The remaining cases get templates with outcome `template_deadline`, and the run ends within 4 hours. | RUN-14 |
| RUN-T13 | A run cancelled during scoring | It is cancelled | It stops, keeps its stored outputs, shows `cancelled`, and has no working paper. | RUN-15 |
| RUN-T14 | Two runs on one submission with different settings | Both finish | Both runs and papers are kept, with different run IDs. | RUN-17 |
| RUN-T15 | A database session scoped to one engagement | It queries another engagement's rows | No rows return. | RUN-18 |
| RUN-T16 | 12 runs in progress during the load test | Status is polled every 10 seconds per run | The 95th and 99th percentiles stay within 300 ms and 1 second. | RUN-19 |
| RUN-T17 | A run interrupted in Mumbai by a failover drill | It resumes in Hyderabad | It finishes with the same decision hash, with no lost or duplicated results. *Recovery: regional failover mid-run* | RUN-20 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. Authentication uses bearer tokens issued per user by an admin command and stored as hashes. This is simpler than an identity service for a small team, and can be swapped later.
2. A token without access to an engagement gets 404, so that engagement IDs cannot be discovered.
3. Files are uploaded through 15-minute upload links straight into the run store, rather than through the API, because `entries.csv` can reach 512 MB.
4. Workers are started per job, with at most 12 run workers. A job whose heartbeat stops for 2 minutes is queued again, up to 3 attempts.
5. At 3 hours 15 minutes, the remaining cases switch to templates, leaving time for the working paper inside 4 hours.
6. Row-level security in PostgreSQL enforces engagement scoping, as well as the checks in code.

---

*Created 2026-10-07.*
