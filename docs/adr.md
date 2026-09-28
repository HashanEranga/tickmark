# Architecture Decision Record

Tickmark's significant design choices, the alternatives we rejected and why, and the assumptions we made where the client has not answered yet. This is deliverable 2 of the [Phase 2 Engagement Handbook](00_Engagement_Handbook.pdf). Requirements come from the [Group 02 Client Brief](Group_02_Client_Brief.pdf); boundaries and acceptance tests come from [scope.md](scope.md).

**Statuses:** *Accepted* is the design we are building. *Proposed* is our choice, pending team confirmation or measurement. *Open* is waiting on the client or on sourced prices. *Superseded* has been replaced, and is kept to show how the design changed.

## The approach in one paragraph

Code decides; agents explain. Rules and statistics score all 400,000 entries and fix the ranking before any model runs. A routed team of agents then investigates only the working-paper entries: a router agent suggests specialists, a routing guard in code guarantees the required ones, three specialists work in parallel from a shared case file, and a writer drafts the justification, which a code verifier checks. The agents are made repeatable by treating every model call as a function of its inputs: its output is validated against a schema, stored under a hash of those inputs, and replayed when the same inputs come back. The orchestrator is a fixed LangGraph graph, built on LangChain, whose branches are decided by code, so agents influence control flow only through stored, validated outputs. The [architecture](diagrams/architecture.md) and [entry investigation](diagrams/entry-investigation.md) diagrams show the same design.

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
| [ADR-015](#adr-015--one-cloud-region-no-multi-cloud-or-multi-region) | One cloud region; no multi-cloud or multi-region | Accepted, pending instructors |
| [ADR-016](#adr-016--processing-and-storage) | Processing and storage | Proposed |
| [ADR-017](#adr-017--orchestrator-built-with-langchain-and-langgraph) | Orchestrator built with LangChain and LangGraph | Accepted |
| [ADR-018](#adr-018--model-provider-and-model-per-role) | Model provider and model per role | Open |
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

### ADR-015 · One cloud region; no multi-cloud or multi-region

**Status:** Accepted, 24 Sep 2026; pending the instructors' answer on the earlier infrastructure slide

**Context:** Client ledgers must stay in the client's region (brief §5). Neither the brief nor the handbook requires multi-cloud, multi-region or zero-downtime deployment.

**Decision:** Deploy everything that touches ledger data (storage, workers, model inference, backups and ledger-bearing telemetry) in one cloud region agreed with the client (assumption A3). Recovery relies on durable storage, idempotent stage outputs and resumable runs rather than a second region.

**Rejected alternatives:**
- *Multi-region active-active:* moves ledger data outside the region unless both regions qualify, and adds cost against the USD 40 ceiling.
- *Multi-cloud:* doubles the platform work with no client requirement behind it. Revisit only if the course requires it (scope §7).
- *A model provider outside the region:* breaks the data constraint.

**Consequences:** A regional outage stops runs until the region recovers, and the runbook says so. The model provider must be available in the region (ADR-018).

### ADR-016 · Processing and storage

**Status:** Proposed, 28 Sep 2026; confirm with the team and the load test

**Context:** Scoring 400,000 rows with rules and statistics is small work for one machine. Most of each run's time goes on model calls, and the peak is 12 concurrent engagements.

**Decision:**
- One worker process per run on a managed container service in the region, scaling to zero outside peak season, with 12 workers at peak.
- PostgreSQL holds run state, case files, the job table and stored agent outputs. Workers claim runs with `SELECT … FOR UPDATE SKIP LOCKED`.
- DuckDB scores each ledger snapshot inside the worker. Criteria are versioned SQL files, which keeps every rule readable and auditable.
- Immutable ledger snapshots (Parquet) and working papers are kept in in-region object storage.
- LangGraph's Postgres checkpointer saves each case's graph state after every node (ADR-017), and stage outputs are written idempotently under run, case and step keys, so a restarted worker resumes without losing or duplicating results.

**Rejected alternatives:**
- *Kubernetes with queue-based autoscaling (KEDA):* real operational load for a peak of 12 jobs. It stays a conditional extension (scope §2).
- *Spark or another distributed engine:* 400,000 rows fit comfortably on one machine.
- *A separate queue service:* one more moving part, when a job table can share a transaction with run state. Revisit if contention appears.
- *A durable-workflow engine such as Temporal:* its replay model fits ADR-010 well, but it adds a stateful platform and raises data-residency questions for the payloads it stores.
- *Serverless functions for the worker:* run-length and concurrency limits vary by provider, and warm state is lost between steps.

**Consequences:** Few moving parts to operate and explain. Rough sizing for the load test to confirm: at most 300 cases × about 10 calls gives 3,000 calls per run; at 8 concurrent calls of about 5 seconds each (assumed), that is roughly 30 minutes, well inside 4 hours. At peak, 12 runs need about 96 concurrent calls, which the provider's in-region quota must allow (ADR-018).

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

**Consequences:** The step ⑤ boxes map one-to-one onto graph nodes, so the diagrams, the code and the traces share names. LangChain's provider integrations make the model-provider choice a configuration change (ADR-018). Because LangChain and LangGraph change quickly, versions stay pinned and the determinism test (ADR-019) guards every upgrade. LangSmith tracing stays switched off, because hosted tracing would send prompts containing ledger text outside the region (ADR-015, ADR-019). Each team member needs to learn LangGraph's state and reducer model.

### ADR-018 · Model provider and model per role

**Status:** Open; needs the client's region (assumption A3) and sourced, dated prices

**Context:** The provider must offer inference inside the client's region, structured outputs, pinned model versions and enough quota for 12 concurrent runs. Every price must carry its source and lookup date (handbook §5).

**Decision so far:** For each role, use the smallest model that passes the agent-quality evaluation: router and specialists first, with the writer possibly larger. Pin exact model versions in the run manifest. Choose the provider once the region is agreed, and record prices with source and date in the cost model.

**Alternatives to compare:** managed model endpoints from the major clouds in the agreed region, and a self-hosted open-weights model in that region, which gives full data control but carries fixed GPU cost that sits idle outside peak season.

**Consequences:** Until this is decided, the cost model shows worst-case cost as a formula of calls, tokens and price rather than a number. Because agents use LangChain's chat-model interface (ADR-017), the final choice is a configuration change rather than a rewrite.

### ADR-019 · Observability and delivery

**Status:** Proposed, 28 Sep 2026

**Context:** The brief lists observability spans per engagement as a cost line, operability carries 10% of the handbook's marks, and the assessors review the GitHub history.

**Decision:**
- OpenTelemetry traces: one trace per run, with spans per stage and per model call, plus metrics for spend, calls, allowance use and cache hits. Spans carry IDs and hashes, never ledger text, and the telemetry backend runs in the region.
- LangSmith tracing stays switched off, because hosted tracing would send prompts containing ledger text outside the region. LangChain callbacks feed the OpenTelemetry traces instead.
- GitHub Actions runs unit tests and a determinism test, in which the decision hash of a fixture ledger must match a checked-in value, then builds and deploys container images.
- Rollback redeploys the previous image. Each run pins its image digest, so a deploy never changes a run in flight.

**Rejected alternatives:**
- *A hosted observability service outside the region, including hosted LangSmith:* ledger-bearing telemetry must stay in the region.
- *Manual deployment:* no audit trail in GitHub and no repeatable rollback.

**Consequences:** The runbook can tie each failure to a span, a metric and a rollback step.

### ADR-020 · Self-set scale extensions from 17 September

**Status:** Superseded by the client brief on 24 Sep 2026

**Context:** Before the brief arrived, we planned continuous monitoring, an interactive review workstation, concurrency for about 200 engagements, multi-cloud and multi-region hosting, and zero-downtime deploys that kept review sessions alive, largely to satisfy an earlier infrastructure slide.

**Why it was replaced:** the brief asks for batch runs with a working paper, sets the peak at 12 concurrent engagements, keeps data in one region and makes USD 40 per engagement the binding constraint. Keeping those extensions would have meant accumulating technology rather than choosing it.

**Kept as:** conditional extensions in scope §2, adopted only if the client or the course requires them.

## Assumptions

Where the client has not told us something, we assume the following and replace each assumption once the answer arrives (scope §7).

| ID | Assumption | Used by | Replace when |
|---|---|---|---|
| A1 | The USD 40 ceiling covers all 3–4 runs of an engagement; replay keeps repeated work free | ADR-010, ADR-014 | The client confirms how re-runs are budgeted |
| A2 | The 4-hour limit applies to each run | ADR-016 | The client confirms |
| A3 | "Our region" is one cloud region agreed with the client, and model inference, backups and ledger-bearing telemetry all count as processing in it | ADR-015, ADR-018, ADR-019 | The client defines the region |
| A4 | An "entry" is one ledger row (posting line), matching the brief's 400,000 rows and 300 reviews; lines of the same journal share a case | ADR-003, ADR-012 | The client confirms the review unit |
| A5 | The working paper is a PDF with a sign-off block (preparer, reviewer, date) and a CSV appendix of evidence | ADR-002 | The client describes a signable working paper |
| A6 | Criteria are equally weighted until the client's risk framework arrives; weights are run settings, so changing them means a re-run, not a code change | ADR-001 | The client shares its risk framework |
| A7 | Normal off-peak volume is unknown, so the cost model shows 1, 4 and 12 engagements a month instead of guessing one figure | ADR-014, ADR-016 | The client gives normal volume |
| A8 | Pass thresholds for precision, recall and false positives are agreed with the client and the domain advisor before the locked evaluation | ADR-004, ADR-007 | The client says what "good enough" means |
| A9 | The handbook governs assessment, so the earlier four-criterion infrastructure slide is not treated as binding | ADR-015, ADR-020 | The instructors answer |
| A10 | A managed model endpoint with structured outputs and pinned versions is available in the agreed region | ADR-018 | The region is agreed and providers are checked |

## How the decisions will be tested

| Decision | Evidence that confirms or overturns it |
|---|---|
| ADR-003 | Identical decision hashes across repeated runs, shuffled input, changed worker counts and retries |
| ADR-007 | Routed team against the fixed chain, code-only routing and a single agent, on the same entries and budget |
| ADR-010 | A repeat run with identical settings makes no model calls and reproduces the working paper |
| ADR-011 | Drafts seeded with a fake evidence ID or a wrong figure are caught before the working paper |
| ADR-013 | Injected narration and cross-engagement queries fail safely |
| ADR-014, ADR-016 | Measured cost stays under the pre-computed worst case, and 12 concurrent full-scale runs each finish inside 4 hours |

## Change log

- **17 Sep 2026:** self-set scope, later superseded (ADR-020).
- **24 Sep 2026:** client brief adopted. ADR-001, ADR-002, ADR-004, ADR-005, ADR-007, ADR-009 and ADR-011 to ADR-015 recorded. The agent design moved from a single model step to a fixed chain (ADR-006), then to the routed team (ADR-007).
- **28 Sep 2026:** this record created. Exact arithmetic (ADR-003), the evidence pack and query catalogue (ADR-008), and replay by input hash (ADR-010) added. Platform proposals (ADR-016 to ADR-019) recorded for team confirmation.
- **28 Sep 2026, later:** the team chose LangChain for the orchestrator. ADR-017 now builds it as a LangGraph graph, the plain-Python proposal became a rejected alternative, and ADR-008, ADR-010, ADR-016, ADR-018 and ADR-019 were updated to match.
