# Scope and Assessment Mapping

Decision record for Tickmark. The [README](../README.md) explains the product; this document defines its delivery boundaries and what the project must prove.

## 1. Authority and current status

The [Phase 2 Engagement Handbook](00_Engagement_Handbook.pdf) defines the shared deliverables, assessment weights and ground rules. The [Group 02 Client Brief v1.0](Group_02_Client_Brief.pdf) (18 Sep 2026) defines the business requirements, constraints, cost unit and peak workload. Where this document and the brief disagree, the brief wins; record every brief change and its effect (§7).

The brief confirms Tickmark's problem: ranking journal entries for ISA 240 testing at a mid-tier audit firm, so that an engagement team knows which of 400,000 entries deserve its roughly 300 reviews, and why. Figures quoted from the brief below are client requirements; anything the brief leaves open is listed as an open decision in §7. No deployment, evaluation result or performance claim is established by this scope document.

### Client constraints (brief §5)

| Constraint | Client requirement | Consequence for Tickmark |
|---|---|---|
| Budget | Under USD 40 of compute and models per 400,000-entry engagement; a hard ceiling and the binding constraint | Rules and statistics process every row; only the working-paper entries, at most 300 per run, reach the agents, and their model calls are counted |
| Runtime | Under 4 hours per engagement, start to finish | Measured end to end at full scale, including at peak concurrency |
| Traceability | Every flag resolves to a source record and a named criterion; an unexplainable flag is worse than none | No flag is emitted without its criterion and source record, and a code verifier rejects any justification that cites evidence the run does not hold |
| Determinism | The same ledger and settings produce the same ranking, and the client's reviewers will check | Ranking is reproducible from a pinned run manifest and is not decided by the agents (§§2, 5) |
| Data | Client ledgers are confidential and stay in the client's region | No cloud provider has a region in Sri Lanka, so the team has chosen Japan, for the client to confirm (§7): storage, processing, the model and telemetry that carry ledger data stay in Japan, across two regions (ADR-015, ADR-018) |

Neither the handbook nor the brief requires multi-cloud, multi-region, Kubernetes, continuous monitoring or zero-downtime deployment. The team has chosen multi-region hosting with a managed model, so that a regional outage in peak season does not stop engagements. No major cloud provider has a region in Sri Lanka, so the team proposes Japan, the only Asian country where Claude on Amazon Bedrock keeps processing in-country across two regions: Tokyo runs the system and Osaka stands by (ADR-015, ADR-018). The client is asked to confirm this (§7, [client log](client-log.md) Q-06). Retain the other capabilities only as conditional extensions (§2).

The routed multi-agent team at step 5 is a team decision rather than a client requirement. The bootcamp is about agentic AI and the project is framed as a multi-agent system, and the brief's sponsor note says where agents beat code: reading narration, weighing context and writing the justification a human will read. The constraints above bound it (§2).

## 2. Product scope

Tickmark supports one audit procedure: **journal entry testing in support of ISA 240**. It is an auditor decision-support tool, not a full audit platform, a fraud determination service or an audit opinion generator.

### Required golden path (brief §§3–4, 11)

For one engagement, run as a batch job with no interactive or conversational interface:

1. An engagement team submits a versioned synthetic ledger of 400,000 entries and run settings: criterion thresholds and weights, plus any engagement parameters such as materiality.
2. Tickmark validates the input, reports rejected records and establishes the population eligible for scoring. It must not silently omit invalid records.
3. Queries, rules and statistics score every accepted entry against the brief's criteria: entries to unrelated or unusual accounts, entries by people who do not normally post, round-sum entries, entries at period end or outside business hours, and entries with weak or missing narration. An entry that trips no criterion is not flagged.
4. Tickmark ranks the flagged entries deterministically and selects up to 300 for the working paper.
5. Only the selected entries reach the routed agent team, which a deterministic orchestrator runs:
   - Related selected entries, such as reversal pairs, amounts split under an approval limit, or one user's entries in one period, are grouped into a single case so that a scheme is investigated once.
   - A router agent suggests which specialists each case needs. A routing guard (code) always adds the specialist that each flagged criterion requires, caps model calls and records the decision.
   - The account-and-amount, poster-and-timing and narration specialist agents run in parallel, only if routed. Each reads the case's evidence pack, which code builds from standard read-only queries, may run a few more queries from a typed catalogue within the case's allowance, and writes a finding with evidence IDs to the shared case file. If a finding needs another specialist's check, the orchestrator allows one follow-up.
   - A writer agent drafts the justification from the findings in a fixed order and shows any disagreement side by side. A code verifier checks that every figure and entry ID in the draft exists in the evidence; a templated justification replaces any draft that fails.

   The run records how many entries reached the agents and how many model calls they made.
6. Tickmark produces a working paper that an engagement reviewer can sign. For each entry it shows the named ISA 240 criterion or criteria that flagged it, the underlying source record, the calculations, the specialists' findings and the justification, with both findings shown wherever specialists disagree. A flag is not a finding of fraud; the auditor concludes.
7. The team refines the criteria and re-runs, typically 3–4 times per engagement. Each re-run is a new versioned run that preserves earlier runs and working papers, and a re-run with unchanged ledger and settings reproduces the same ranking.
8. Each run completes in under 4 hours and the engagement stays under USD 40, demonstrated at full scale rather than projected. Results remain retrievable after a service restart.

This path must be deployed, reachable and demonstrated end to end while the brief's peak of 12 concurrent engagements is running. One complete path takes priority over multiple partial modes or infrastructure demonstrations (handbook §§2–3).

### In scope for the first complete delivery

- Batch journal-entry risk ranking over full-scale synthetic ledgers. The simulator has two business profiles, retail and wholesale trading; the brief does not fix an industry.
- Versioned synthetic ledger generation at 400,000 entries with a realistic chart of accounts, posting patterns, period-end behaviour and seeded anomalies (brief §7), plus input validation and documented rejection behaviour.
- A rules-and-statistics pre-filter over every entry covering the brief's criteria, deterministic ranking and a working paper of up to 300 entries.
- A routed agent team for the working-paper entries only: a router agent, three specialist agents (account and amount, poster and timing, narration) and a writer agent, run by a deterministic orchestrator with case grouping, a code-built evidence pack and typed query catalogue, a routing guard, a code verifier, templated fallback, a fixed model allowance per case, and per-run counts of entries and model calls (brief §§6, 11).
- Versioned re-runs with refined criteria, persisted run provenance and reproducibility checks.
- Access control and engagement-scoped access checks; test that one engagement cannot read another's data.
- Storage and processing of ledger data in Japan across two AWS regions, Tokyo primary and Osaka standby, including model calls and any telemetry that carries ledger content, with a rehearsed failover between them.
- A deployment and automated verification path through GitHub, with observable failures and a documented rollback procedure.
- All six handbook deliverables and the evaluation, load and commercial evidence defined below.

Rules and statistics decide flags and ranking; the agents do not. The router chooses who investigates a case, never what is flagged or in which order, and cannot drop a specialist that a flagged criterion requires; if the router fails, the case gets the required specialists only. Specialists share a case file instead of talking to each other, and at most one follow-up round is allowed. The agents' tools are typed, read-only queries from a fixed catalogue, scoped to one engagement; agents never write their own SQL or reach other systems. Each case has a fixed model allowance, set before the run, that covers its worst case. When the agents' model is unavailable or a draft fails verification, each flag still carries its criterion, source record and calculations, with a templated justification.

### Conditional extensions

Promote an extension into required scope only when the client or course requires it, or a recorded, evidence-based decision justifies it, and only if it stays within the budget and data constraints:

- **Agent narration assessment in the ranking:** letting the narration specialist's assessment change flags or ranking would extend its work beyond the 300 selected entries. It needs an ADR, a cap on candidates that keeps the engagement under USD 40, and record-and-replay of each assessment so identical settings still give an identical ranking.
- **Queue-based autoscaling:** scale workers with pending work, including to zero outside peak season, if that lowers normal-volume cost. Consider KEDA only if Kubernetes is justified, after comparing simpler worker and deployment options.
- **Multi-cloud:** a second provider in the same country, only if the course requires it or availability justifies the cost against the USD 40 ceiling; the same decision output and tested recovery apply. Multi-region hosting is already in scope (ADR-015).
- **Zero-downtime releases:** a batch system needs no session-preserving deploys beyond the required guarantee that a deploy or worker loss neither loses nor duplicates an in-flight run's results (§5).
- **Narration clustering with embeddings:** an optional cost line in the brief (§6); adopt it only if it measurably improves the weak-narration criterion within budget.

IT services and deadline-weighted scheduling remain stretch goals. No extension displaces missing evaluation, load-test, cost-model or runbook evidence.

### Out of scope

- Concluding whether fraud occurred: Tickmark ranks and the auditor concludes (brief §9). Also other audit procedures, audit opinions or financial statement conclusions.
- A conversational interface or interactive review workstation; this is a batch system with a report (brief §§4, 9). Recording auditor dispositions inside Tickmark is not required.
- Continuous monitoring and ERP integration, including automatically blocking or approving journal entries; the brief asks for neither.
- Sending every row, or unfiltered rows, to a language model.
- A free-running orchestrator agent or open-ended agent-to-agent conversation; routing and follow-ups stay bounded.
- Claims of ISA 240 compliance or proven real-world fraud detection based only on synthetic tests.
- Real client ledgers or real personal data; all demonstration and evaluation data must be synthetic, with generation documented.
- Autonomous auditor conclusions, or agent- or model-determined scores, criterion triggers or rankings unless the conditional extension above is adopted.

### Domain validation

Request the client's risk framework, which the brief offers on request (brief §10), and use it to weight the criteria. Seek validation from a practising audit manager for simulator scenarios, rule definitions and evaluation interpretation. This is a planned validation activity, not an approval already obtained. Each criterion records its audit rationale, applicable ISA 240 reference and edition, assumptions and limitations. Distinguish requirements in the standard from project-designed heuristics; do not imply that ISA 240 prescribes a particular scoring formula or review budget.

## 3. Assessment mapping

The weights below come from handbook §3. Whether the earlier four-criterion infrastructure slide also applies is an open question for the instructors (§7).

| Weight | Assessment area | Tickmark evidence |
|---|---|---|
| 25% | Does it work | A full 400,000-entry engagement completed live in under 4 hours and under USD 40 while 12 engagements run concurrently, ending in a signable working paper |
| 20% | Engineering judgement | ADRs explaining significant choices, rejected alternatives, assumptions and trade-offs, above all where the boundary between rules and agents sits, and whether routing to specialists beats a fixed chain or code-only routing |
| 20% | Evidence | Named evaluation cases, pass rate, precision and recall against seeded anomalies, agent justification quality, known misses and load-test results that can be reproduced |
| 15% | Commercial thinking | Sourced and dated unit prices, cost per 400,000-entry engagement against the USD 40 ceiling, normal/peak monthly costs, margins, break-even and the point where Tickmark stops being cheaper than manual selection |
| 10% | Operability | Runbook, useful observability, behaviour when the agents' model or another dependency is down, and demonstrated rollback, recovery and regional failover |
| 10% | Working as a team | Elicited requirements, client questions, named component ownership and GitHub history showing each member built something |

Infrastructure earns its place by supporting a client requirement and producing evidence, not by adding technologies. The handbook explicitly warns that a demonstration without evaluation, scale evidence and cost arithmetic is insufficient.

## 4. Mandatory delivery artefacts

These six artefacts are required by handbook §2; their completion is separate from feature completion.

| Artefact | Minimum acceptance evidence |
|---|---|
| Working system | Reachable deployment and a repeatable demonstration of the complete golden path under peak load: a full 400,000-entry ledger in under 4 hours and under USD 40, demonstrated rather than projected |
| Architecture decision record | Significant choices, rejected alternatives and reasons; assumptions where client information is missing; include deployment topology and region, the rules-to-agents boundary, the routing design and agent roles, persistence and processing design; kept in [adr.md](adr.md) |
| Cost model | Cost per 400,000-entry engagement against the USD 40 ceiling; monthly cost at normal and peak volume; margin at three price points; break-even; the point where it stops being cheaper than today's manual selection; how many entries reached the agents, how many model calls they made, and why; source and lookup date for every unit price, with unavailable prices explicitly treated as assumptions |
| Evaluation report | Named ordinary, awkward and hostile cases; expected and actual outcomes; pass rate; precision and recall against seeded anomalies; agent justification quality; known misses and whether their causes are understood |
| Load or scale test | Evidence against the brief's peak of 12 concurrent 400,000-entry engagements; completion time per engagement against the 4-hour limit; p95/p99 latency, error rate and cost under load; workload, environment and measurement method recorded |
| Operations runbook | What breaks, how it is detected, service/data impact, cost implications, recovery steps and rollback procedure |

Do not label an artefact complete solely because a template exists. It needs actual results, decisions or rehearsed operational steps.

## 5. Evaluation and reproducibility

### Reproducibility contract

The client requires that two runs over the same ledger with the same settings produce the same ranking, and its reviewers will check (brief §5). Given the same immutable accepted ledger snapshot and run manifest, Tickmark must produce the same:

- score per entry;
- triggered rules and calculations;
- evidence references;
- ordered ranking and shortlist of up to 300 entries.

The result must be identical across repeated runs, worker counts, retries and input delivery order, and in both regions: a run resumed in the standby region after a failover gives the same result. If multiple providers are delivered, the same contract applies across them.

Each run manifest pins the input snapshot, schema, criterion thresholds and weights, other engagement settings such as materiality, reference data, rule bundle, scoring version, case-grouping rules, the criterion-to-specialist routing table, the query catalogue, model and prompt versions for each agent, simulator seed where applicable and execution image. Store amounts as integer minor units and compute scores in fixed-point arithmetic, so no result depends on summation order or worker count; define timestamp handling and stable entry IDs. Stable entry IDs break ranking ties. Inputs and canonically serialized decision outputs receive SHA-256 hashes; exclude variable operational metadata such as execution timestamps from the decision hash.

The routed agent team investigates and justifies the working-paper entries; it does not determine scores, criteria or ranking. Validate each routing decision and agent output against its schema, and store it with its model, provider and prompt provenance under a hash of the call's exact input, role, prompt, model, schema and catalogue versions; label generated text as such. Whenever the same hash recurs, reuse the stored output, so re-runs repeat the same routing and wording without spending the model budget again; a zero temperature setting is not relied on for repeatability. An output that fails its schema gets one repair attempt, then the template. Findings reach the writer in a fixed order, so the order in which parallel specialists finish cannot change the draft. When the agents fail, the structured evidence and a templated justification stand in. Agent failure or fallback may change wording, never the decision or evidence.

The acceptance test compares a canonical decision hash across repeated runs, shuffled input, changed worker counts and worker retries. Test deployment interruption, recovery and regional failover without missing or duplicating logical results. Cross-provider equality and uninterrupted releases are additional tests only if those capabilities are delivered.

### Named functional and adversarial cases

The evaluation report must give each case a stable ID, expected result and actual outcome. At minimum, cover:

| Class | Cases to define and run |
|---|---|
| Ordinary | Full 400,000-entry ledger; each brief criterion flags its seeded cases; every flag resolves to a source record and named criterion; an entry flagged on several criteria gets every required specialist; every justification passes the verifier or falls back to the template; working paper produced; a criteria change creates a separate run |
| Awkward | Empty and fewer-than-300 flagged populations; tied scores; duplicate IDs; missing fields; unbalanced journals; amount precision; period-boundary timestamps; specialists that disagree; related entries grouped into one case |
| Hostile | Malformed or oversized uploads; unauthorised engagement access; repeated submissions; adversarial narration and prompt injection aimed at the agents, including attempts to query another engagement's data |
| Recovery | Worker termination and retry; persistence/queue outage where applicable; rollback; regional failover mid-run; model-provider outage; router failure falling back to the required specialists; a case exhausting its model allowance |

Agree validation and rejection policy before testing awkward inputs; do not treat every unusual journal as fraud. Report passed cases divided by executed cases, and show failed, blocked and unexecuted cases separately. Record known causes, unknown causes and limitations rather than omitting failures.

### Detection quality

Use a versioned ledger simulator with separate development and locked test scenarios, generating every evaluation ledger at the full 400,000 entries; results on a few thousand rows do not count (brief §7). Ground truth distinguishes fraud typology, injected scenario and affected journal entries. Freeze rules and scoring choices before the locked evaluation, and record any later tuning as a new evaluation version. The [evaluation plan](evaluation-plan.md) sets out the ledgers, metrics and pass thresholds, and runs the frozen pipeline on two public datasets as a separately reported external check.

**Benchmark (brief §§1–2):** 400,000 entries with a review capacity of about 300, both set by the client. The brief uses rows and entries interchangeably; the team counts an entry as one journal line ([client log](client-log.md) Q-09).

Report:

- entry recall@300;
- scenario recall@300, with a scenario counted as hit when at least one affected entry is shortlisted;
- precision@300;
- median first-hit rank, identifying undetected scenarios separately;
- recall by fraud typology.

For populations below 300, use the actual shortlist size and label the effective cutoff. Report undefined metrics explicitly when their denominator is zero. Compare with random, value-based and simple rule-based baselines using the same population and review budget. Store simulator version, seed, configuration and ledger hash; document synthetic-data generation and do not expose ground-truth labels to scoring.

Synthetic results demonstrate performance on the defined scenarios, not general real-world fraud-detection accuracy. The brief has not said how many false positives a team will tolerate (brief §10), so the pass thresholds are a team decision, set before the locked evaluation and sent to the client to confirm (client log Q-02). Do not invent a passing target after seeing the results.

Evaluate the agent team separately from detection: report the verifier pass rate, the template fallback rate, how often the narration specialist agrees with the seeded weak-narration cases, how often a specialist the router added finds something the required ones missed, and a sampled review of justifications by the domain advisor. Compare the routed team with a fixed chain, code-only routing and a single-agent baseline on the same entries and budget; if the router rarely adds a useful finding, choose code-only routing.

## 6. Load, operations and commercial evidence

### Load and scale

The brief's peak is 12 concurrent engagements of 400,000 entries each, January to March, with 3–4 re-runs per engagement, and every run must finish in under 4 hours (brief §§4–5). The brief does not state normal off-peak volume; obtain it before calculating normal monthly cost. Record ledger size, runs started per time interval, concurrent engagements, re-run mix and measurement environment.

Measure the complete path rather than only worker throughput:

- end-to-end completion time for every engagement in the peak test, against the 4-hour limit;
- p95/p99 latency for pipeline stages and model calls, with separate definitions and sample counts;
- throughput, queue wait/backlog where applicable and time to drain the peak;
- error rate with an explicit denominator, rejected inputs and failed/retried jobs;
- cost per engagement during the measured test against the USD 40 ceiling, and the entries and model calls that reached the agents;
- output correctness, identical rankings for identical settings, and engagement isolation under load.

Confirm error-rate and recovery targets with the client before recording a pass. Demonstrate that all 12 concurrent engagements finish inside the limit and that none is starved. Add autoscaling and multi-provider tests only when those capabilities enter required scope; a model-provider outage and a regional failover are already required recovery cases (§5).

### Operations

Instrument run IDs and engagement context, processing duration, backlog, failures/retries, dependency health, spend per engagement, and each routing decision, agent step and tool call, without exposing ledger content; telemetry that does carry ledger data stays in Japan. The runbook must connect failure symptoms to detection, diagnosis, recovery, rollback and cost impact, including a run on course to breach the 4-hour or USD 40 limit. Rehearse at least a worker interruption, a model-provider outage, another critical dependency outage, a regional failover and failback, and deployment rollback; preserve or safely resume accepted work and completed results. Record any downtime honestly rather than assuming zero-downtime delivery.

### Cost and pricing

The unit is one audit engagement of 400,000 entries, and USD 40 of compute and models per engagement is a hard ceiling and the binding constraint (brief §§5–6). Cost is driven by how few entries reach a model, not by how fast the system runs. Confirm whether the 3–4 re-runs share that ceiling (§7); until then, show the cost both ways.

The cost model should separate fixed monthly costs from volume-dependent costs and cover the brief's line items: rules and statistical pre-filter compute over all 400,000 rows, agent model calls for routing, specialist investigation, follow-ups and justification (priced per token, including the 10% premium for keeping calls in Japan), embeddings if used, database and query, application compute, and observability spans per engagement, plus networking/egress, cross-region replication, queues, retries and idle or redundant infrastructure such as the standby region. State retention and utilisation assumptions; show normal and peak monthly workloads separately rather than treating the January–March peak as the whole year.

For each volume scenario, calculate:

- total monthly cost = fixed monthly cost + variable cost for that workload;
- effective cost per engagement = total monthly cost / completed engagements;
- revenue and margin at three stated price points, defining margin as `(revenue - modelled cost) / revenue` and stating which costs are included;
- break-even engagements = fixed monthly cost / `(price per engagement - variable cost per engagement)` where linear assumptions apply and contribution is positive; otherwise report no finite break-even under those assumptions or use a tier-aware calculation;
- the point at which Tickmark stops being cheaper than the people who select entries today, by comparing cost per engagement with 2–3 auditor-days of selection at a sourced rate.

State how many entries reached the agents per run, how many model calls they made, and why those numbers are what they are (brief §11). Because each case's model allowance is fixed before the run, also state the worst-case run cost (cases × allowance) next to the measured cost. Every external unit price needs its source and lookup date. List missing prices as assumptions, distinguish free credits from sustainable pricing and reconcile estimates with measured load-test usage. The standby region, and any multi-cloud redundancy, must show its incremental cost against the USD 40 ceiling.

## 7. Client decisions and team working

The handbook and the brief both assess the questions asked as well as the system built (handbook §§3–4, brief §10). Because the demo and evaluation use only synthetic ledgers, the team decides each question below and builds on its decision. Send those the [client log](client-log.md) marks for confirmation through the standing client channel, and do not mark any confirmed without a recorded response.

Brief v1.0 answers the problem, the unit (one 400,000-entry engagement), batch operation with a working-paper output, review capacity (about 300), peak volume (12 concurrent engagements, January–March), re-runs (3–4 per engagement), the 4-hour and USD 40 limits, and the determinism, traceability and data requirements.

| Question | Why it matters |
|---|---|
| Obtain the client's risk framework: which ISA 240 criteria it weights most heavily, and how materiality sets thresholds (brief §10) | Sets criterion weights, and therefore the ranking |
| Agree what "good enough" means, including how many false positives a team will tolerate (brief §10) | Sets the evaluation pass thresholds |
| Agree what a working paper must contain to be signed off (brief §10) | Defines the golden path's output and its acceptance |
| Learn what engagement teams do today and why they distrust the last tool the firm bought (brief §10) | Shows which failures would make the client reject Tickmark |
| Confirm whether the 3–4 re-runs share the USD 40 ceiling, and whether the 4-hour limit applies to each run | Decides whether each run gets USD 40 or a third to a fifth of it |
| Confirm whether ledgers may be stored and processed outside Sri Lanka, in Japan as proposed, including model inference, backups and telemetry; whether AWS is acceptable; and how long ledgers and results are retained | Decides the hosting country, the model provider, observability tooling and storage cost |
| Confirm whether an entry is a journal line or a journal header | Changes the ledger schema, the simulator and every metric |
| Obtain normal off-peak volume | Required for normal monthly cost and break-even |
| Ask the instructors whether the earlier four-criterion infrastructure slide applies alongside the handbook | Decides whether multi-cloud or zero-downtime is mandatory, which the data constraint then limits |
| Assign component owners and arrange domain review | Enables accountable delivery and credible audit rationale |

Keep a dated record of questions, team decisions, replies and scope changes in the [client log](client-log.md). For each change, record its effect on the golden path, evidence, cost and priorities. The handbook says clients respond within one working day; raise a client blocker the same day rather than silently proceeding.

Each team member must own a named component and contribute implementation through GitHub. The brief names three members; assign each a concrete boundary, for example ledger simulator, criteria engine and ranking; the routed agent team, verifier and working paper; or platform, evaluation, load testing and cost model. No assignments are confirmed here. Retain a GitHub history showing individual contributions. Disclose AI assistance and ensure each owner can explain what they shipped (handbook §6).

## 8. Delivery order and completion gate

1. **Decide and confirm:** record the team's decision on each §7 question in the [ADR](adr.md) and the [client log](client-log.md), send those marked for confirmation in week one, and assign owners. Only a reply that contradicts a decision changes the plan.
2. **Complete one path:** build and deploy the full-scale batch path, from ledger ingestion through rules-and-statistics filtering, ranking and the routed agent team to the signable working paper, with durable evidence.
3. **Prove it:** run the named evaluation and reproducibility cases, run 12 concurrent full-scale engagements, show every run under 4 hours and every engagement under USD 40, finish the cost arithmetic and rehearse recovery, rollback and regional failover. Capture ADRs and client decisions throughout delivery, not retrospectively.
4. **Extend only when justified:** add client- or course-mandated capabilities before optional ones, keep every extension within the budget and data constraints, and repeat affected tests and cost calculations.

Delivery is complete when the brief's six-point definition of done is met (brief §11), the deployed golden path works under peak load, all six artefacts contain evidence, failures and limitations are disclosed, AI assistance is disclosed, and ownership/client-engagement evidence is visible in GitHub. An impressive architecture alone does not satisfy this gate.

## 9. Positioning

- Say **"reduces the review population,"** never "replaces the auditor." Tickmark ranks; the auditor concludes.
- Lead with the complete workflow and measured evidence for assessors, and the audit-review problem for industry.
- Describe unbuilt infrastructure as proposals, and quote the brief's figures as client requirements until measured results replace them.
- Code decides what is flagged and in what order; the agents explain why.
- Claim reproducibility for deterministic decisions and evidence, not agent wording.
- Treat the 300-entry working paper as the client's review capacity, not a guarantee that no further audit work is necessary.

---

*Last revised 2026-09-30 against the Phase 2 Engagement Handbook, the Group 02 Client Brief v1.0 (18 Sep 2026), the [ADR](adr.md), the [client log](client-log.md) and the [evaluation plan](evaluation-plan.md).*
