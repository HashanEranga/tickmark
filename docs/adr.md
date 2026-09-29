# Architecture Decision Record

Tickmark's significant design choices, the alternatives we rejected and why, and the assumptions we made where the client has not answered yet. This is deliverable 2 of the [Phase 2 Engagement Handbook](00_Engagement_Handbook.pdf). Requirements come from the [Group 02 Client Brief](Group_02_Client_Brief.pdf); boundaries and acceptance tests come from [scope.md](scope.md).

**Statuses:** *Accepted* is the design we are building. *Proposed* is our choice, pending team confirmation or measurement. *Open* is waiting on the client or on sourced prices. *Superseded* has been replaced, and is kept to show how the design changed.

## The approach in one paragraph

Code decides; agents explain. Rules and statistics score all 400,000 entries and fix the ranking before any model runs. A routed team of agents then investigates only the working-paper entries: a router agent suggests specialists, a routing guard in code guarantees the required ones, three specialists work in parallel from a shared case file, and a writer drafts the justification, which a code verifier checks. The agents are made repeatable by treating every model call as a function of its inputs: its output is validated against a schema, stored under a hash of those inputs, and replayed when the same inputs come back. The orchestrator is a fixed LangGraph graph, built on LangChain, whose branches are decided by code, so agents influence control flow only through stored, validated outputs. Everything that touches client ledgers, including the model, stays in Japan across two AWS regions: Tokyo runs the system and Osaka stands by. The [architecture](diagrams/architecture.md) and [entry investigation](diagrams/entry-investigation.md) diagrams show the same design.

## Decisions at a glance

| ID | Decision | Status |
|---|---|---|
| [ADR-001](#adr-001--code-decides-flags-and-ranking-agents-only-explain) | Code decides flags and ranking; agents only explain | Accepted |
| [ADR-002](#adr-002--batch-runs-that-end-in-a-signable-working-paper) | Batch runs that end in a signable working paper | Accepted |
| [ADR-003](#adr-003--exact-reproducible-ranking-under-a-pinned-run-manifest) | Exact, reproducible ranking under a pinned run manifest | Accepted |
| [ADR-004](#adr-004--full-scale-synthetic-ledger-with-seeded-anomalies) | Full-scale synthetic ledger with seeded anomalies | Accepted |
| [ADR-005](#adr-005--agents-see-only-the-working-paper-entries) | Agents see only the working-paper entries | Accepted |
| [ADR-006](#adr-006--fixed-agent-chain) | Fixed agent chain | Superseded by ADR-007 |
| [ADR-007](#adr-007--routed-agent-team-with-a-code-routing-guard) | Routed agent team with a code routing guard | Accepted, pending evaluation |
| [ADR-008](#adr-008--code-built-evidence-pack-plus-a-typed-query-catalogue) | Code-built evidence pack plus a typed query catalogue | Accepted |
| [ADR-009](#adr-009--shared-case-file-one-follow-up-disagreements-shown) | Shared case file, one follow-up, disagreements shown | Accepted |
| [ADR-010](#adr-010--repeatable-agents-typed-outputs-replayed-by-input-hash) | Repeatable agents: typed outputs replayed by input hash | Accepted |
| [ADR-011](#adr-011--code-verifier-with-a-template-fallback) | Code verifier with a template fallback | Accepted |
| [ADR-012](#adr-012--related-entries-grouped-into-one-case) | Related entries grouped into one case | Accepted |
| [ADR-013](#adr-013--narration-is-untrusted-agent-tools-are-read-only) | Narration is untrusted; agent tools are read-only | Accepted |
| [ADR-014](#adr-014--fixed-model-allowance-per-case) | Fixed model allowance per case | Accepted |
| [ADR-015](#adr-015--multi-region-in-japan-tokyo-primary-osaka-standby) | Multi-region in Japan: Tokyo primary, Osaka standby | Accepted, pending client |
| [ADR-016](#adr-016--processing-and-storage) | Processing and storage | Proposed |
| [ADR-017](#adr-017--orchestrator-built-with-langchain-and-langgraph) | Orchestrator built with LangChain and LangGraph | Accepted |
| [ADR-018](#adr-018--claude-on-amazon-bedrock-kept-in-japan) | Claude on Amazon Bedrock, kept in Japan | Accepted, models chosen by evaluation |
| [ADR-019](#adr-019--observability-and-delivery) | Observability and delivery | Proposed |
| [ADR-020](#adr-020--self-set-scale-extensions-from-17-september) | Self-set scale extensions from 17 September | Superseded by the client brief |

## Decisions

### ADR-001 · Code decides flags and ranking; agents only explain

**Status:** Accepted, 24 Sep 2026

**Context:** The binding constraint is USD 40 of compute and models per 400,000-entry engagement, and the client's reviewers will re-run ledgers and expect the same ranking (brief §5). The sponsor's note says to find candidates with queries, rules and statistics, and to use agents only where they beat code.

**Decision:**
- Named criteria, written as rules and statistics, score every accepted entry. An entry that trips no criterion is not flagged.
- Code ranks the flagged entries and selects up to 300. No model output can change a flag, a score or the order.
- Agents investigate and explain the selected entries (ADR-005 to ADR-013).

**Rejected alternatives:**
- *A language model reading every row:* exceeds the budget by orders of magnitude (brief §5), and its output cannot be repeated exactly.
- *A model re-ranking the top candidates:* breaks the determinism check unless every judgement is recorded and replayed, and pushes more entries through a model. It stays a conditional extension (scope §2).
- *An unsupervised anomaly model, such as an isolation forest, making the flag decision:* its scores do not resolve to a named criterion, which the brief requires for every flag. It could later feed one statistical criterion if seeded and versioned.

**Consequences:** Detection quality is bounded by the criteria, so precision and recall depend on rule design and the client's risk framework. Decision determinism is testable without any model. The agents must prove their value through explanation quality, not detection.

### ADR-002 · Batch runs that end in a signable working paper

**Status:** Accepted, 24 Sep 2026

**Context:** The brief describes batch work (one engagement, 3–4 re-runs, 12 concurrent engagements at peak), puts a conversational interface out of scope and asks for a working paper an engagement reviewer can sign.

**Decision:** Each request creates a versioned batch run through the Run API, and each run ends in a working paper. Re-runs never overwrite earlier runs. There is no interactive review screen or chat interface; auditor dispositions are recorded on the signed working paper, not in Tickmark.

**Rejected alternatives:**
- *An interactive review workstation* (our own idea from 17 September, ADR-020): the brief asks for a batch system with a report.
- *Continuous monitoring of a live journal feed:* not requested, and it adds streaming infrastructure with no client value.
- *A conversational assistant over the ledger:* explicitly out of scope in the brief.

**Consequences:** There is no session state to protect during deploys, so a deploy only has to avoid losing or duplicating an in-flight run's results. The working-paper format is an open client question (assumption A5).

### ADR-003 · Exact, reproducible ranking under a pinned run manifest

**Status:** Accepted, 28 Sep 2026 (refines the reproducibility contract of 24 Sep)

**Context:** The same ledger and settings must give the same ranking across repeated runs, worker counts, retries and input order (scope §5). Floating-point sums computed in parallel can differ in their last digits depending on how the work is split, and a near-tie can then flip.

**Decision:**
- Amounts are stored as integer minor units, and scores use fixed-point arithmetic with a documented rounding rule, so no result depends on summation order.
- Ties break on stable entry IDs.
- Each run manifest pins the ledger snapshot hash, schema, criterion thresholds and weights, rule bundle, case-grouping rules, routing table, query catalogue, each agent's prompt and model version, and the execution image digest.
- Canonically serialised decision outputs get a SHA-256 decision hash that excludes timestamps and other operational metadata.

**Rejected alternatives:**
- *Floating-point scores compared within a tolerance:* the client's reviewers compare rankings, not tolerances, and near-ties would still flip.
- *Single-threaded execution as the guarantee:* slower, and it fails silently the first time someone raises the worker count.

**Consequences:** The acceptance test compares decision hashes across repeated runs, shuffled input, changed worker counts and retries. Any change to scoring is a new rule-bundle version and a new run, never an edit to an old one.

### ADR-004 · Full-scale synthetic ledger with seeded anomalies

**Status:** Accepted, 24 Sep 2026

**Context:** Real client data is out of scope. The brief asks for a full-scale synthetic ledger with seeded anomalies, so that precision and recall can be measured, and does not accept a system proven only on a few thousand rows (brief §7).

**Decision:** Build a versioned ledger simulator that generates 400,000 entries with a realistic chart of accounts (retail first), posting patterns and period-end behaviour, and seeds known anomaly scenarios. Development and locked test scenarios are kept separate. Ground-truth labels are stored apart from the ledger and are never visible to scoring or to the agents.

**Rejected alternatives:**
- *Public or sample ledgers:* none exist at this scale with a known answer for every row.
- *Small ledgers extrapolated to full size:* the brief rejects results that are projected rather than demonstrated.
- *Tuning and evaluating on the same scenarios:* overfits; the locked test set prevents it.

**Consequences:** Results show performance on our scenarios, not real-world fraud detection, and the evaluation report says so. The domain advisor reviews the scenarios for realism.

### ADR-005 · Agents see only the working-paper entries

**Status:** Accepted, 24 Sep 2026

**Context:** Model calls drive the cost, and the brief asks us to report how many entries reached a model and why that number is what it is (brief §11).

**Decision:** Only the entries selected under ADR-001, at most 300 per run, reach the agents. Each run records how many entries and model calls reached them.

**Rejected alternatives:**
- *Agents on every flagged candidate, often thousands:* cost grows with rule looseness, and the extra justifications are never read, because only 300 fit in the working paper.
- *One summary call for the whole working paper:* cheaper, but each entry's justification would lose its evidence links.

**Consequences:** The answer to "why this number" is short: at most 300 per run, fewer when related entries share a case (ADR-012), and fewer again on re-runs that reuse stored outputs (ADR-010).

### ADR-006 · Fixed agent chain

**Status:** Superseded by ADR-007 on 24 Sep 2026

**Context:** After the brief arrived, the first design gave one model a single job: wording the justification. That left no multi-agent system in a project framed as one, so the design became a fixed chain of an investigator agent, a narration reviewer and a writer for every selected entry.

**Why it was replaced:** every entry passed through every agent in the same order whether it needed them or not, which spent budget on irrelevant checks, and no agent specialised in a criterion. ADR-007 routes each case to the specialists it needs.

**Kept as:** a comparison baseline in the agent evaluation (ADR-007).

### ADR-007 · Routed agent team with a code routing guard

**Status:** Accepted, 24 Sep 2026; review after the locked evaluation

**Context:** Flagged entries trip different criteria, often several at once. The sponsor's note says agents are better than code at reading narration, weighing context and writing the justification a reviewer reads.

**Decision:**
- Three specialist agents cover the brief's five criteria: *account and amount* (unrelated or unusual accounts, round sums), *poster and timing* (people who do not normally post, period end, outside business hours) and *narration* (weak or missing narration).
- A router agent reads each case and suggests the specialists it needs. A routing guard in code always adds the specialist each flagged criterion requires, accepts only suggestions from the fixed menu, caps calls and records the decision. If the router fails, the case gets the required specialists only.
- A writer agent turns the findings into the justification.

**Rejected alternatives:**
- *A single agent with tools:* simplest, but one prompt must cover every criterion. Kept as an evaluation baseline.
- *The fixed chain (ADR-006):* runs every agent on every entry.
- *Routing in code only:* deterministic and free. It is the fallback if the router does not earn its place.
- *A free-running supervisor agent that plans and loops until satisfied:* unbounded cost and runtime against hard USD 40 and 4-hour limits, and its control flow cannot be replayed.
- *Voting or debate between agents or models:* multiplies cost and hides disagreement that the reviewer should see (ADR-009).

**Consequences:** The router costs one model call per case and earns its place only when a specialist it adds finds something the required ones missed. The evaluation compares the routed team with the fixed chain, code-only routing and a single agent on the same entries and budget. If the router rarely adds a useful finding, measured against a threshold agreed before the locked evaluation, switch to code-only routing and record that in a new ADR.

### ADR-008 · Code-built evidence pack plus a typed query catalogue

**Status:** Accepted, 28 Sep 2026

**Context:** Specialists need context: the poster's history, how the account is normally used, typical amounts, timing against the close calendar and related entries. If agents wrote their own queries, results would depend on what each agent happened to ask, injected narration could steer the queries, and every query round would cost a model call.

**Decision:**
- When a case opens, code runs a fixed set of read-only queries for its flagged criteria and stores the results in the case file as the *evidence pack*, each fact with an evidence ID.
- A specialist that needs more can call a few queries (three to start, tuned with evaluation) from a typed catalogue: named functions with validated parameters, exposed to the agent as LangChain tools, such as a poster's postings to an account over twelve months. Code runs them, scoped to the engagement, and adds the results to the case file.
- Specialists never write SQL and cannot reach other systems.

**Rejected alternatives:**
- *Agents generating SQL:* an injection risk, results depend on generated text, and verification gets harder.
- *The evidence pack only, with no extra queries:* the most deterministic and cheapest option, but it stops agents from following a lead. It is what happens when a case's allowance runs out.
- *Unbounded tool loops:* cost and runtime become unpredictable.

**Consequences:** Most cases need one model call per specialist, because the evidence pack answers the common questions. Every fact a justification cites carries an evidence ID the verifier can check (ADR-011). The catalogue is versioned and pinned in the run manifest.

### ADR-009 · Shared case file, one follow-up, disagreements shown

**Status:** Accepted, 24 Sep 2026

**Context:** Many cases need two or three specialists at once. Sometimes one specialist's finding raises a question for another, and specialists can disagree.

**Decision:**
- Specialists run in parallel and never talk to each other. Each reads the case file and writes a structured finding with evidence IDs.
- A finding may request another specialist's check. The routing guard sends at most one follow-up per case, inside the allowance.
- Findings reach the writer in a fixed criterion order, so the order in which specialists finish cannot change the draft.
- When findings conflict, the writer shows both with their evidence and marks the entry "specialists disagree". No agent picks a winner and the ranking does not change; the auditor concludes.

**Rejected alternatives:**
- *Agent-to-agent conversation:* unbounded, costly and not replayable.
- *Sequential hand-offs between specialists:* slower, and it brings back the fixed chain.
- *A majority vote, or a judge agent settling conflicts:* hides the disagreement an auditor most needs to see.

**Consequences:** Disagreement becomes a visible signal on the working paper. The single follow-up bounds cost; if evaluation shows follow-ups are often needed, widen the evidence pack rather than adding rounds.

### ADR-010 · Repeatable agents: typed outputs replayed by input hash

**Status:** Accepted, 28 Sep 2026 (formalises the record-and-replay rule of 24 Sep)

**Context:** Language models are not reliably repeatable, even at temperature zero, yet a re-run with the same ledger and settings should reproduce the working paper and should not pay for the same work twice.

**Decision:**
- Every agent output (router suggestion, finding, justification) is a typed object validated against a JSON schema. Free text appears only in designated fields.
- Every model call is keyed by a SHA-256 hash of its canonical input: agent role, prompt version, model version, schema and catalogue versions, and the case evidence it sees. When a key has been seen before, the stored output is returned instead of calling the model.
- The orchestrator is a fixed LangGraph graph (ADR-017) whose branching edges are plain code. The only branches that depend on agents are the router's suggestion, a follow-up request and the verifier's result, and each of those comes from a stored, validated output.
- An output that fails its schema gets one repair attempt if the case's allowance has room, keyed and stored the same way. A second failure, or no room left, falls back to the template (ADR-011).

**Rejected alternatives:**
- *Temperature zero on its own:* reduces variation but does not remove it.
- *No replay:* re-runs would change the wording and spend the budget again.
- *Caching by entry ID only:* would wrongly reuse an output after the evidence or the prompt changed.

**Consequences:** A re-run with identical settings makes no model calls and reproduces the working paper, apart from generation timestamps; this is also why re-runs fit the budget (assumption A1). An environment without the stored outputs may word justifications differently. The decision and evidence are unaffected, and scope §9 claims reproducibility only for those.

### ADR-011 · Code verifier with a template fallback

**Status:** Accepted, 24 Sep 2026

**Context:** The client says a flag it cannot explain is worse than no flag, and every flag must resolve to a source record and a named criterion (brief §5).

**Decision:** Before a justification reaches the working paper, code checks that:
- every cited evidence ID exists in the case file;
- every number and date in the text matches a value in the cited evidence;
- every flagged criterion is addressed;
- no sentence concludes that fraud occurred.

A failed draft gets one repair attempt using the verifier's errors (ADR-010), then a templated justification built from the criteria and evidence. The template is also used when the model is unavailable.

**Rejected alternatives:**
- *A language-model judge as the verifier:* not repeatable, and it can make the same mistakes it is meant to catch. It may help offline in evaluation instead.
- *No verification:* the brief's traceability requirement would rest on trust.

**Consequences:** Every flag always ships with its criterion, source record and calculations. The verifier pass rate and template fallback rate are reported in the evaluation (scope §5).

### ADR-012 · Related entries grouped into one case

**Status:** Accepted, 24 Sep 2026

**Context:** Some anomaly scenarios span several entries: a round-sum entry reversed after period end, amounts split to stay under an approval limit, or one user's unusual entries within one period.

**Decision:** Before routing, code groups related selected entries into a case using versioned rules pinned in the run manifest: lines of the same journal, reversal pairs, amounts split under an approval limit, and one user's entries within one period. The team investigates each case once, and every entry's justification points to the shared case. Grouping never changes the ranking.

**Rejected alternatives:**
- *One case per entry:* repeats work and misses the story that links the entries.
- *Grouping by a language model or by embedding similarity:* not repeatable. Embeddings stay a conditional extension for narration (scope §2).

**Consequences:** Schemes cost fewer model calls and get one explanation. The grouping rules need their own evaluation cases (scope §5).

### ADR-013 · Narration is untrusted; agent tools are read-only

**Status:** Accepted, 24 Sep 2026

**Context:** Narration is typed by client staff and goes straight into agent prompts, so it can carry instructions. Engagements must never see each other's data.

**Decision:** The Run API authenticates every caller and scopes each run to one engagement. Narration and other ledger text reach agents as quoted data, never as instructions. Agent tools are the read-only query catalogue (ADR-008), always scoped by code to the run's engagement. Agents have no write access, no network access and no way to change the ranking, and their outputs are schema-validated and verified (ADR-010, ADR-011).

**Rejected alternatives:**
- *Relying on prompt wording alone to resist injection:* a hope, not a control.
- *Giving agents general database or web access:* unnecessary for the task and a path for data to leak.

**Consequences:** The hostile evaluation cases (scope §5) include injected narration and attempts to query another engagement's data, and both must fail safely.

### ADR-014 · Fixed model allowance per case

**Status:** Accepted, 24 Sep 2026; values are set in the cost model

**Context:** USD 40 per engagement is a hard ceiling (brief §§5–6), and a run's cost must not depend on luck or timing.

**Decision:**
- Each case gets an allowance fixed before the run: a maximum number of model calls and a token limit per call. It covers the worst case: the router, all three specialists with their extra queries, one follow-up, and the writer with one repair.
- The worst-case run cost (cases × allowance × sourced price) is computed before the run and reported next to the measured cost.
- If the worst case would not fit, shrink the allowance, with fewer extra queries or a smaller model for specialists, before dropping any required specialist.

**Rejected alternatives:**
- *A single run-wide budget that stops when spent:* which entries get agent justifications would then depend on processing order and timing.
- *No cap:* cost would only be known afterwards.

**Consequences:** Cost is bounded before any model call is made. Allowance values are sized in the cost model under the stricter reading of the budget (assumption A1).

### ADR-015 · Multi-region in Japan: Tokyo primary, Osaka standby

**Status:** Accepted, 30 Sep 2026 (team decision; replaces "Everything in Sri Lanka" of 29 Sep, which had replaced "one cloud region" of 24 Sep); pending client confirmation ([client log](client-log.md) Q-06, Q-07)

**Context:** Client ledgers must stay in the client's region (brief §5). The team wants multi-region hosting, so that a regional outage during the January–March peak does not stop engagements, and a managed model rather than a self-hosted one. None of AWS, Azure or Google Cloud has a region in Sri Lanka, so a multi-region design has to keep ledgers in another country that the client accepts (assumption A3). As of 30 Sep 2026, Japan is the only Asian country where Amazon Bedrock keeps current Claude models' processing in-country across two regions, Tokyo and Osaka. In Mumbai, Hyderabad, Singapore, Seoul and Jakarta, Bedrock offers Claude only with global routing, which may process a call anywhere.

**Decision:**
- Everything that touches ledger data stays in Japan: storage, workers, model inference, backups, and logs or traces that contain ledger data. Code, synthetic ledgers and CI can run anywhere, because none of them is client data.
- Tokyo (`ap-northeast-1`) is the primary region and runs all work. Osaka (`ap-northeast-3`) is a warm standby: it keeps continuously replicated copies of the run store and container images, and has the Run API and workers deployed but scaled to zero.
- If Tokyo fails, the runbook promotes the Osaka database replica, starts the Run API and workers there and points the API's DNS name at Osaka. Each in-flight run resumes from its last checkpoint (ADR-016, ADR-017).
- Recovery targets, to be confirmed by a failover drill: at most about a minute of lost writes, and service back within 30 minutes, so an interrupted run still finishes inside 4 hours.
- Model calls use Bedrock's Japan inference profile, which sends each call to Tokyo or Osaka and never outside Japan (ADR-018), so a model outage in one region needs no failover.

**Rejected alternatives:**
- *Everything in Sri Lanka with a self-hosted model* (the 29 Sep design): there is no second region to fail over to, the GPU is a fixed cost, and open-weights models may write weaker justifications. It remains the fallback if the client requires Sri Lanka (Q-06, Q-08).
- *Mumbai and Hyderabad:* the nearest regions, but Bedrock offers Claude there only with global routing, so model calls could be processed outside India.
- *Singapore paired with another Asian region:* the same problem.
- *Sydney and Melbourne:* Bedrock's Australia profile meets the same rules, but it is farther from Sri Lanka with no offsetting advantage.
- *Active-active in both regions:* doubles the always-on cost and needs a database that accepts writes in both regions, for a peak of only 12 runs.
- *Multi-cloud:* a second provider doubles the platform work, and Google's multi-region Claude endpoints cover only the US and the EU. It stays a conditional extension (scope §2).
- *Bedrock's global endpoint:* cheaper and more available, but it may process calls outside Japan.

**Consequences:** Ledgers leave Sri Lanka, so this design stands only if the client accepts Japan (Q-06). The standby adds a fixed monthly cost, mostly the database replica, which the cost model shows against the USD 40 ceiling (scope §6), and model calls cost 10% more than global routing. A failover can lose the last moments of replicated work. Idempotent stage outputs and replay (ADR-010) make repeating that work safe: the ranking cannot change (ADR-003), and only the wording of repeated agent steps may differ. The runbook covers failover and failback, and both are rehearsed.

### ADR-016 · Processing and storage

**Status:** Proposed, 28 Sep 2026; updated 30 Sep for two AWS regions in Japan; confirm with the team and the load test

**Context:** Scoring 400,000 rows with rules and statistics is small work for one machine. Most of each run's time goes on model calls, and the peak is 12 concurrent engagements. Tokyo runs everything and Osaka stands by (ADR-015).

**Decision:**
- One worker process per run, as a container task on Amazon ECS with Fargate: up to 12 at peak and none when idle. Osaka runs none until a failover.
- PostgreSQL on Amazon RDS holds run state, case files, the job table and stored agent outputs, with a cross-region read replica in Osaka. Workers claim runs with `SELECT … FOR UPDATE SKIP LOCKED`.
- DuckDB scores each ledger snapshot inside the worker. Criteria are versioned SQL files, which keeps every rule readable and auditable.
- Immutable ledger snapshots (Parquet) and working papers are kept in Amazon S3 in Tokyo and replicated to Osaka, and container images are replicated the same way.
- LangGraph's Postgres checkpointer saves each case's graph state after every node (ADR-017), and stage outputs are written idempotently under run, case and step keys, so a restarted worker in either region resumes without losing or duplicating results.

**Rejected alternatives:**
- *Kubernetes with queue-based autoscaling (KEDA):* real operational load for a peak of 12 jobs. It stays a conditional extension (scope §2).
- *Spark or another distributed engine:* 400,000 rows fit comfortably on one machine.
- *A separate queue service:* one more moving part, when a job table can share a transaction with run state. Revisit if contention appears.
- *A durable-workflow engine such as Temporal:* its replay model fits ADR-010 well, but it adds another stateful platform to run and replicate across two regions.
- *Serverless functions:* a worst-case run takes about an hour, beyond a function's time limit (15 minutes on AWS Lambda).
- *Aurora Global Database:* faster, managed failover, but it costs more at this size. Switch to it if the failover drill misses its targets.

**Consequences:** Few moving parts to operate and explain, and nothing runs in Osaka except the database replica until a failover. Rough sizing for the load test to confirm, assuming about 3,000 input and 600 output tokens and 5 seconds per call:
- At most 300 cases × about 10 calls gives 3,000 calls per run. At 8 concurrent calls, a run's model work takes about 30 minutes.
- 12 such runs would need about 3.5 million input and 0.7 million output tokens per minute. Bedrock allows 2 million input tokens per minute by default, and grants up to 5 million input and 0.5 million output on request without special approval (ADR-018).
- Capping each run at 4 concurrent calls halves the demand, which then fits the quotas granted on request, and still finishes a worst-case run in about an hour.

### ADR-017 · Orchestrator built with LangChain and LangGraph

**Status:** Accepted, 28 Sep 2026 (the team's decision; replaces a plain-Python proposal made the same day)

**Context:** The team is building in Python and chose LangChain for the orchestrator. The orchestration must stay bounded and replayable (ADR-007, ADR-009, ADR-010): a fixed set of steps, branches decided by code, and no open-ended agent loops.

**Decision:**
- The orchestrator is a LangGraph graph, LangGraph being LangChain's library for stateful workflows. Each box in step ⑤ is a node: open case, router, routing guard, the three specialists, follow-up, writer, verifier and template.
- Every conditional edge is a plain Python function over validated state. The router's suggestion reaches the rest of the graph only through the routing-guard node.
- Specialists fan out in parallel with LangGraph's `Send`. The writer node sorts findings into the fixed criterion order before prompting, so the order in which branches finish cannot matter (ADR-009).
- Agents are LangChain chat models with structured output (Pydantic schemas). The query catalogue is bound to specialists as LangChain tools with validated arguments (ADR-008), with at most one tool round per specialist.
- Each agent node checks the replay store (ADR-010) before calling the model, so repeatability rests on our own input-hash key rather than on how the framework builds its cache keys.
- LangGraph's Postgres checkpointer saves each case's state after every node, so a restarted worker resumes the case where it stopped (ADR-016).
- LangChain and LangGraph versions are pinned in the lockfile and baked into the execution image, which the run manifest pins (ADR-003).

**Rejected alternatives:**
- *A plain Python state machine* (the earlier proposal): full control, but we would rebuild the checkpointing, parallel fan-out and model-provider integrations that LangGraph and LangChain already provide.
- *LangChain's prebuilt tool-calling agent loops* (ReAct-style): the model decides which tools to call and when to stop, which conflicts with bounded, replayable runs (ADR-007, ADR-009).
- *LangChain chains on their own:* fine for single calls, but awkward for conditional routing, parallel fan-out and resuming after a crash.
- *Conversation-driven multi-agent frameworks* (AutoGen- or CrewAI-style): control flow emerges from agents talking to each other, which conflicts with ADR-009 and ADR-010.

**Consequences:** The step ⑤ boxes map one-to-one onto graph nodes, so the diagrams, the code and the traces share names. LangChain talks to Claude on Bedrock like any other chat model, so changing the model or region is a configuration change (ADR-018). Because LangChain and LangGraph change quickly, versions stay pinned and the determinism test (ADR-019) guards every upgrade. LangSmith tracing stays switched off, because hosted tracing would send prompts containing ledger text outside Japan (ADR-015, ADR-019). Each team member needs to learn LangGraph's state and reducer model.

### ADR-018 · Claude on Amazon Bedrock, kept in Japan

**Status:** Accepted, 30 Sep 2026 (replaces "Self-hosted open-weights models in Sri Lanka" of 29 Sep); the model for each role is chosen by the agent-quality evaluation

**Context:** Model calls carry ledger text, so they must stay in Japan (ADR-015). As of 30 Sep 2026, Amazon Bedrock serves current Claude models through its Messages API with a Japan inference profile that sends each call to Tokyo or Osaka, at a 10% premium over global routing. Anthropic's own API pins inference only to the US or globally, and Google's multi-region Claude endpoints cover only the US and the EU. Bedrock's Claude endpoint offers neither provider-side structured outputs nor the Message Batches API. Every price must carry its source and lookup date (handbook §5).

**Decision:**
- Call Claude through Bedrock's Messages API with the Japan inference profile, so every call is processed in Tokyo or Osaka and a model outage in one region is absorbed without a failover.
- For each role, use the smallest model that passes the agent-quality evaluation. Start with Claude Haiku 4.5 for the router and specialists and Claude Sonnet 5 for the writer, and compare them with Sonnet 5 in every role. Before pinning a model, confirm that the Japan profile serves it.
- Pin each role's model ID and the inference profile in the run manifest (ADR-003).
- Pass each agent's output schema as a tool definition through LangChain's structured-output support, because the endpoint has no native structured-output mode. Code still validates every output and replays it by input hash (ADR-010).
- Cache the fixed part of each prompt (instructions, schema and query catalogue) with prompt caching.
- If the Japan profile fails, calls retry with backoff within the run's time limit, and then the case falls back to the template (ADR-011). Calls never fall back to the global endpoint.

**Rejected alternatives:**
- *Self-hosted open-weights models in Sri Lanka* (the 29 Sep design): a fixed GPU cost, and possibly weaker justifications. It returns only if the client requires Sri Lanka (ADR-015).
- *Bedrock's global endpoint:* no premium and the best availability, but calls may be processed outside Japan.
- *Anthropic's API or Claude Platform on AWS:* inference can be pinned only to the US or globally.
- *Google Vertex:* its multi-region endpoints cover only the US and the EU, and its single-region endpoints serve only Claude Sonnet 4.6 and older.
- *Batch pricing:* the Message Batches API and its discount are not available on Bedrock.

**Consequences:** Model cost is per call again, with no fixed GPU. At Anthropic's list prices plus the 10% premium (list prices checked 29 Sep 2026: Haiku 4.5 at USD 1 and Sonnet 5 at USD 2 per million input tokens, USD 5 and USD 10 per million output tokens), the worst case with Haiku 4.5 in every role is about USD 20 per run: 3,000 calls of about 3,000 input and 600 output tokens (ADR-014, assumption A1). Sonnet 5 as the writer adds about USD 4, and prompt caching lowers both. The cost model replaces these estimates with Bedrock's own dated prices and measured tokens. At peak, Bedrock's per-minute token quotas limit concurrency more than runtime does (ADR-016). A first spike must confirm that LangChain works against this endpoint with short-lived AWS credentials.

### ADR-019 · Observability and delivery

**Status:** Proposed, 28 Sep 2026; updated 30 Sep for two AWS regions in Japan

**Context:** The brief lists observability spans per engagement as a cost line, operability carries 10% of the handbook's marks, and the assessors review the GitHub history.

**Decision:**
- OpenTelemetry traces: one trace per run, with spans per stage and per model call, plus metrics for spend, calls, allowance use and cache hits. Spans carry IDs and hashes, never ledger text, and the telemetry backend runs in Japan.
- Health checks on the Run API, workers, database and model calls raise an alarm that starts the failover runbook (ADR-015).
- LangSmith tracing stays switched off, because hosted tracing would send prompts containing ledger text outside Japan. LangChain callbacks feed the OpenTelemetry traces instead.
- GitHub Actions runs unit tests and a determinism test, in which the decision hash of a fixture ledger must match a checked-in value, then builds container images and deploys them to both regions through a short-lived AWS role, with no stored keys. Tests use synthetic fixtures only, so no ledger data passes through GitHub.
- Rollback redeploys the previous image. Each run pins its image digest, so a deploy never changes a run in flight.

**Rejected alternatives:**
- *A hosted observability service outside Japan, including hosted LangSmith:* telemetry that carries ledger data must stay in Japan.
- *Manual deployment:* no audit trail in GitHub and no repeatable rollback.

**Consequences:** The runbook can tie each failure to a span, a metric, and a rollback or failover step.

### ADR-020 · Self-set scale extensions from 17 September

**Status:** Superseded by the client brief on 24 Sep 2026

**Context:** Before the brief arrived, we planned continuous monitoring, an interactive review workstation, concurrency for about 200 engagements, multi-cloud and multi-region hosting, and zero-downtime deploys that kept review sessions alive, largely to satisfy an earlier infrastructure slide.

**Why it was replaced:** the brief asks for batch runs with a working paper, sets the peak at 12 concurrent engagements, keeps data in one region and makes USD 40 per engagement the binding constraint. Keeping those extensions would have meant accumulating technology rather than choosing it.

**Kept as:** conditional extensions in scope §2, adopted only if the client or the course requires them. Multi-region hosting came back on 30 Sep 2026 as a team decision, inside one country (ADR-015).

## Assumptions

Where the client has not told us something, we assume the following and replace each assumption once the answer arrives (scope §7). The [client log](client-log.md) tracks each question, its presumed answer and whether the client has confirmed it.

| ID | Assumption | Used by | Replace when |
|---|---|---|---|
| A1 | Each run is capped at USD 30–40, and the design targets about USD 20 worst case per run, so an engagement's 3–4 runs stay under USD 40 even if the ceiling is per engagement; replay keeps repeated work free (team decision, 29 Sep 2026) | ADR-010, ADR-014, ADR-018 | The client confirms how re-runs are budgeted (Q-05) |
| A2 | The 4-hour limit applies to each run | ADR-016 | The client confirms (Q-05) |
| A3 | "Our region" can be a country the client approves, not only Sri Lanka, and the team proposes Japan: storage, processing, the model, backups and ledger-bearing telemetry all stay in Japan, across Tokyo and Osaka (team decision, 30 Sep 2026; replaces "Sri Lanka only" of 29 Sep) | ADR-015, ADR-016, ADR-018, ADR-019 | The client confirms the region (Q-06) |
| A4 | An "entry" is one journal line, matching the brief's 400,000 rows and 300 reviews; lines of the same journal share a case (team decision, 29 Sep 2026) | ADR-003, ADR-012 | The client confirms the review unit (Q-09) |
| A5 | The working paper follows the 4-page sample shared on 29 Sep 2026: a PDF with a sign-off block (preparer, reviewer, manager) and CSV appendices | ADR-002 | The client describes a signable working paper (Q-03) |
| A6 | Criteria are equally weighted until the client's risk framework arrives; weights are run settings, so changing them means a re-run, not a code change | ADR-001 | The client shares its risk framework (Q-01) |
| A7 | Normal off-peak volume is unknown, so the cost model shows 1, 4 and 12 engagements a month instead of guessing one figure | ADR-014, ADR-016, ADR-018 | The client gives normal volume (Q-10) |
| A8 | Pass thresholds for precision, recall and false positives are agreed with the client and the domain advisor before the locked evaluation; the client log holds the presumed targets | ADR-004, ADR-007 | The client says what "good enough" means (Q-02) |
| A9 | The handbook governs assessment, so the earlier four-criterion infrastructure slide is not treated as binding | ADR-015, ADR-020 | The instructors answer (I-01) |
| A10 | The client accepts AWS, and the team runs the system and the capstone demo in its own AWS account, with Claude on Bedrock enabled in Tokyo and Osaka, on synthetic ledgers at list prices (no course credits assumed) | ADR-015, ADR-016, ADR-018 | The client (Q-07) and the instructors (I-02) answer |

## How the decisions will be tested

| Decision | Evidence that confirms or overturns it |
|---|---|
| ADR-003 | Identical decision hashes across repeated runs, shuffled input, changed worker counts and retries |
| ADR-007 | Routed team against the fixed chain, code-only routing and a single agent, on the same entries and budget |
| ADR-010 | A repeat run with identical settings makes no model calls and reproduces the working paper |
| ADR-011 | Drafts seeded with a fake evidence ID or a wrong figure are caught before the working paper |
| ADR-013 | Injected narration and cross-engagement queries fail safely |
| ADR-014, ADR-016 | Measured cost stays under the pre-computed worst case, and 12 concurrent full-scale runs each finish inside 4 hours |
| ADR-015 | A failover drill: a run interrupted in Tokyo resumes in Osaka within the recovery targets, with the same decision hash and no lost or duplicated results. A deployment check shows that no ledger data, model call or trace leaves Japan |
| ADR-018 | Haiku 4.5 and Sonnet 5 are compared role by role on the agent-quality cases, and 12 concurrent runs stay within Bedrock's quotas and finish inside 4 hours |

## Change log

- **17 Sep 2026:** self-set scope, later superseded (ADR-020).
- **24 Sep 2026:** client brief adopted. ADR-001, ADR-002, ADR-004, ADR-005, ADR-007, ADR-009 and ADR-011 to ADR-015 recorded. The agent design moved from a single model step to a fixed chain (ADR-006), then to the routed team (ADR-007).
- **28 Sep 2026:** this record created. Exact arithmetic (ADR-003), the evidence pack and query catalogue (ADR-008), and replay by input hash (ADR-010) added. Platform proposals (ADR-016 to ADR-019) recorded for team confirmation.
- **28 Sep 2026, later:** the team chose LangChain for the orchestrator. ADR-017 now builds it as a LangGraph graph, the plain-Python proposal became a rejected alternative, and ADR-008, ADR-010, ADR-016, ADR-018 and ADR-019 were updated to match.
- **29 Sep 2026:** the team set the region to Sri Lanka, where no major cloud provider or managed Claude endpoint can keep processing, and presumed answers to the open client questions.
- **30 Sep 2026:** ADR-015 first put everything in Sri Lanka, with self-hosted open-weights models in ADR-018. The team rejected that the same day in favour of multi-region hosting with a managed model: ADR-015 now uses two AWS regions in Japan, Tokyo primary and Osaka standby, and ADR-018 uses Claude on Amazon Bedrock through its Japan profile. ADR-016, ADR-017, ADR-019 and ADR-020 were updated to match, and the Sri Lankan design became the fallback if the client requires Sri Lanka. Assumptions A1, A3, A4 and A10 hold the presumed answers, which are tracked in the new [client log](client-log.md).
