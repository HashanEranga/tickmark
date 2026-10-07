# Spec 00 · Ground rules

**Status:** Agreed, 7 Oct 2026 · **Owner:** the whole team · **Applies to:** every spec, pull request and line of code

These are the rules that hold everywhere in Tickmark. Every component spec builds on them and does not repeat them. Each rule restates a decision from the [ADR](../docs/adr.md), which explains why it was made. This file says what must hold and how we check it.

## How the documents fit together

The client brief sets the requirements, and [scope.md](../docs/scope.md) sets the boundaries. The [ADR](../docs/adr.md) records each decision and why it was made, each spec says what one component must do, and the code says how it does it. When two of them disagree, the one earlier in that list wins, and the later one is fixed in the same pull request.

## How specs are written

1. There is one spec per component, in build order (see [the specs to write](#the-specs-to-write)), named `specs/NN-name.md`.
2. Each spec has six parts:
   - purpose;
   - in and out of scope;
   - requirements;
   - data shapes;
   - acceptance tests;
   - open questions.

   A template will follow.
3. Each requirement has an ID made of its spec's prefix and a number, such as `RNK-04`. It is one testable sentence in EARS form (the Easy Approach to Requirements Syntax):
   - always: "Tickmark shall …";
   - on an event: "When …, Tickmark shall …";
   - in a state: "While …, Tickmark shall …";
   - for something unwanted: "If …, then Tickmark shall …";
   - for an option: "Where …, Tickmark shall …".
4. Each requirement has at least one acceptance test, with an ID such as `RNK-T04`, and each test names the requirements it checks. Tests use synthetic data only. The evaluation reuses them as its named ordinary, awkward, hostile and recovery cases (scope §5).
5. Each spec moves through four statuses:
   - *Draft* until a teammate who did not write it approves its pull request into `develop`;
   - *Agreed* once that pull request merges;
   - *Built* once its code merges;
   - *Verified* once its acceptance tests pass in CI.
6. The spec comes first. If the code needs different behaviour, change the spec in a pull request first. Code pull requests cite the requirement IDs they implement.
7. After the locked evaluation, changing a requirement that affects flags, scores or the ranking starts a new evaluation version (scope §5).
8. A ground rule changes only together with the ADR it comes from, in the same pull request.

## The rules

### 1. Code decides, agents explain

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-01 | Tickmark shall decide every flag, score and ranking position in code, with rules and statistics. No model output shall change them. | ADR-001 | The decision hash is the same with the agents switched on, switched off or failing |
| GR-02 | Tickmark shall send only the selected entries, at most 300 per run, to the agents. | ADR-005 | Each run's record of entries sent to the agents |
| GR-03 | Tickmark shall run agents only as nodes of the fixed orchestrator graph, where code decides every branch. There shall be no open-ended agent loops and no agent-to-agent conversation. | ADR-007, ADR-009, ADR-017 | Graph review in the pull request; a failed router leaves the case with only the specialists its criteria require |

### 2. Same input, same result

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-04 | When the accepted ledger snapshot and the run manifest are the same, Tickmark shall produce the same decision hash, whatever the input order, the worker count, the retries or the region. | ADR-003 | The determinism test on every pull request, and repeated full-size runs in the evaluation |
| GR-05 | Tickmark shall store amounts as integer minor units and compute scores in fixed-point arithmetic with a documented rounding rule. Floating-point numbers shall not decide flags, scores or order. | ADR-003 | Unit tests on amounts and scores at their boundaries |
| GR-06 | Tickmark shall break ranking ties on stable entry IDs. | ADR-003 | The tied-scores case |
| GR-07 | Tickmark shall take every date, threshold and seed that a decision uses from the ledger or the run manifest, never from the clock, the environment or unseeded randomness. | ADR-003 | The determinism test, repeated with a different system clock |
| GR-08 | Tickmark shall pin in each run manifest everything that can change a decision or an agent output. If an item is missing, Tickmark shall not start the run. | ADR-003, ADR-018, ADR-022 | A run with an incomplete manifest is refused |
| GR-09 | When any manifest item changes, Tickmark shall create a new run and leave earlier runs and working papers unchanged. | ADR-002, ADR-003 | The criteria-change case |
| GR-10 | When a model call's input hash has been seen before, Tickmark shall return the stored output instead of calling the model. | ADR-010 | A repeat run makes no model calls and reproduces the working paper |

The run manifest pins the ledger snapshot hash, the schema, the criterion thresholds and weights, other engagement settings such as materiality, the rule bundle, the case-grouping rules, the routing table, the query catalogue, each agent's prompt, model ID and inference profile, the standard editions cited (ADR-022), the simulator seed where one was used, and the execution image digest.

### 3. Every flag explained

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-11 | Tickmark shall link every flag to its source record and to at least one named criterion, C1 to C5, with that criterion's ISA 240 and SLAuS 240 references. | Brief §11, ADR-022 | Every flag in the working paper resolves |
| GR-12 | If a justification cites a figure, date or entry ID that is not in the case's evidence, then Tickmark shall reject the draft. After one repair attempt, it shall use the template. | ADR-011 | Drafts seeded with a fake evidence ID or a wrong figure are caught |
| GR-13 | Tickmark shall not state or imply in any justification that fraud occurred. | ADR-011 | A verifier rule, plus the domain advisor's sampled review |
| GR-14 | If an input record is invalid, then Tickmark shall reject it, with a reason, in the run's rejection report. It shall never drop a record silently. | Scope §2 | The awkward cases: missing fields, duplicate IDs, unbalanced journals |

### 4. Agents and model calls

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-15 | Tickmark shall validate every agent output against its JSON schema before using it. | ADR-010 | Schema tests, and the malformed-output case |
| GR-16 | Tickmark shall pass ledger text, including narration, to agents only as quoted data, never as instructions. | ADR-013 | The injected-narration cases fail safely |
| GR-17 | Tickmark shall give agents only the read-only query catalogue, scoped by code to the run's engagement. Agents shall not write SQL, change data or reach the network. | ADR-008, ADR-013 | A query aimed at another engagement's data fails |
| GR-18 | Tickmark shall fix each case's model allowance before the run starts. If the allowance runs out, then the case shall finish without further model calls. | ADR-014 | The allowance-exhaustion case |
| GR-19 | If the model is unavailable or a draft fails verification, then Tickmark shall still deliver the flag with its criterion, source record, calculations and a templated justification. | ADR-011, ADR-018 | The model-outage case |
| GR-20 | Tickmark shall choose the model provider by configuration only: Ollama locally and Claude on Bedrock in the cloud. The graph, schemas and verifier shall be the same in both. | ADR-021 | The Bedrock smoke test after each merge into `develop` |

### 5. Data and access

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-21 | Tickmark shall use synthetic ledgers only, in every environment, with no real client ledgers or personal data. | Scope §2, ADR-004 | Pull request review |
| GR-22 | In the cloud, Tickmark shall keep everything that touches ledger data in India: storage, workers, model calls through Bedrock's India profile, backups, and any logs or traces that contain ledger data. Model calls shall never fall back to Bedrock's global endpoint. | ADR-015, ADR-018 | The deployment check for ADR-015 |
| GR-23 | Tickmark shall put IDs and hashes in telemetry, never ledger text. | ADR-019 | A run with marker text in its narration leaves no marker in logs or traces |
| GR-24 | Tickmark shall authenticate every Run API caller and scope each run to one engagement. No engagement shall read another's data. | ADR-013 | The unauthorised-access case |
| GR-25 | Tickmark shall keep ground-truth labels apart from the ledgers, where neither scoring nor the agents can read them. | ADR-004 | Scoring and the agents run without access to the labels |
| GR-26 | The repository shall hold no ledgers, secrets, credentials or client PDFs, and services shall use AWS roles instead of stored keys. | ADR-018, ADR-019 | Pull request review and `.gitignore` |

### 6. Limits and recovery

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-27 | Tickmark shall finish each run in under 4 hours and keep each engagement under USD 40 of compute and model calls, with 12 engagements running at once. | Brief §5 | The load test and the cost model |
| GR-28 | Tickmark shall record each run's worst-case model cost, computed before the first model call. It shall also record the entries sent to the agents, the model calls made, the tokens used and the measured spend. | ADR-005, ADR-014 | The run record, compared with the cost model |
| GR-29 | Tickmark shall write every stage output idempotently under its run, case and step keys, so that a worker restart, a retry or a regional failover neither loses nor duplicates results. | ADR-015, ADR-016 | The worker-termination and failover drills |
| GR-30 | Tickmark shall finish each in-flight run on the image it started with, and a deploy shall not interrupt it. | ADR-019 | A deploy during a running engagement |

### 7. Honest evaluation

| ID | Rule | From | Checked by |
|---|---|---|---|
| GR-31 | The team shall freeze the rules and scoring before the locked evaluation. Any later change shall start a new evaluation version. | Scope §5 | The evaluation report's version history |
| GR-32 | The team member who seeds a scenario shall not write the criteria that detect it. | ADR-004 | Authorship in the GitHub history |
| GR-33 | Reports shall show failed, blocked and unexecuted cases and known misses. They shall claim neither ISA 240 or SLAuS 240 compliance nor real-world fraud detection. | Scope §5, ADR-022 | Review of the evaluation report |

## Words we use

One word means one thing everywhere: in specs, code, logs and the working paper.

| Term | Meaning |
|---|---|
| Engagement | One client audit. It has its own ledgers and runs, and no access to any other engagement's. |
| Ledger snapshot | An accepted, unchangeable copy of one ledger, identified by its hash. |
| Entry | One journal line, the unit that is scored and ranked (assumption A4). |
| Journal | The entries posted together. Their debits and credits must balance. |
| Criterion | One of the brief's five warning signs: C1, an unrelated or unusual account; C2, a person who does not normally post; C3, a round sum; C4, period end or outside business hours; C5, weak or missing narration. |
| Flag | The mark on an entry that trips at least one criterion. Only flagged entries are ranked. |
| Run | One versioned batch job over one ledger snapshot and one run manifest. |
| Run manifest | The pinned list of everything that can change a run's result (GR-08). |
| Decision hash | The SHA-256 hash of a run's decisions: each entry's score, its flags and calculations, its evidence references, and the ranking. It leaves out timestamps. |
| Case | One or more related selected entries, investigated together (ADR-012). |
| Case file | A case's evidence pack, findings and agent outputs, kept in the run store. |
| Evidence ID | The ID of one fact in a case file. Justifications must cite these. |
| Allowance | A case's fixed maximum number of model calls, and its token limit per call. |
| Replay store | The stored agent outputs, each keyed by the hash of its call's input. |
| Template | A justification built by code from the criteria and the evidence, used when no agent draft can be. |
| Working paper | A run's output for the engagement reviewer to sign: a PDF plus CSV files. |

## The specs to write

In build order. Owners are set by the team.

| # | File | Prefix | Covers | Main ADRs |
|---|---|---|---|---|
| 01 | `01-ledger-format.md` | FMT | Journal-line fields, run settings, validation and the rejection report | ADR-003, ADR-016 |
| 02 | `02-ledger-generator.md` | GEN | Business profiles, normal entries, seeded scenarios, innocent look-alikes and labels | ADR-004 |
| 03 | `03-rules-and-ranking.md` | RNK | Criteria C1 to C5, scoring, ranking and the selection of up to 300 entries | ADR-001, ADR-003, ADR-022 |
| 04 | `04-cases-and-evidence.md` | CAS | Case grouping, the evidence pack and the query catalogue | ADR-008, ADR-012 |
| 05 | `05-agent-team.md` | AGT | Router, routing guard, specialists, writer, allowances and replay | ADR-007, ADR-009, ADR-010, ADR-014, ADR-017, ADR-018 |
| 06 | `06-verifier-and-templates.md` | VER | Verifier checks, the repair attempt and template justifications | ADR-011 |
| 07 | `07-working-paper.md` | WPR | PDF and CSV output, and the sign-off block | ADR-002, ADR-022 |
| 08 | `08-run-api-and-worker.md` | RUN | Run API, access checks, job table, worker, run store and resuming | ADR-002, ADR-013, ADR-016 |
| 09 | `09-deployment-and-operations.md` | OPS | Local Docker setup, AWS in Mumbai and Hyderabad, CI/CD, telemetry, failover and rollback | ADR-015, ADR-019, ADR-021 |
| 10 | `10-evaluation.md` | EVL | The evaluation harness, baselines, dataset converters and the load test | ADR-004 and the [evaluation plan](../docs/evaluation-plan.md) |

---

*Created 2026-10-07.*
