# Client log

A dated record of the open questions, the team's decision on each, and any reply from the client or the instructors. Tickmark's demo and evaluation use only synthetic ledgers that the team generates, so the team decides every answer and builds on it; nothing waits on a reply (team decision, 30 Sep 2026). Questions marked "Send" still go out as "please confirm", because the handbook assesses the questions we ask (handbook §§3–4), and [scope.md](scope.md) §7 requires this record. Design decisions that depend on these answers are in the [ADR](adr.md).

## How to read it

- **Team decision:** the answer the team has decided and builds on. Affected specs carry its ADR assumption ID, for example [A3]. "Send" says whether we still ask the client or the instructors to confirm it.
- **Confirmed:** the client or the instructors agreed. Record the date and their wording.
- **Changed:** the reply differs from the team's decision. Record what we updated in the change record below.

Rules:

- Record something as the client's answer only if the client actually said it.
- When an answer arrives, update the question, the matching ADR assumption and any affected spec in the same pull request, and add a row to the change record.
- Fill in "Sent" with the date each question goes to the client. The client replies within one working day (handbook §4); raise anything that blocks us the same day (handbook §6).
- If the team revises a decision before a reply arrives, say so in the question's status line and record why in the ADR change log.

## Brief versions

| Version | Dated | Changes |
|---|---|---|
| v1.0 | 18 Sep 2026 | First issue; no changes since |

## Summary

| ID | Topic | Status | Send | Sent | Assumption |
|---|---|---|---|---|---|
| [Q-01](#q-01--risk-framework) | Risk framework | Team decision | Yes | – | A6 |
| [Q-02](#q-02--what-good-enough-means) | What "good enough" means | Team decision | Yes | – | A8 |
| [Q-03](#q-03--working-paper-format) | Working-paper format | Team decision | Yes | – | A5 |
| [Q-04](#q-04--current-practice-and-the-last-tool) | Current practice and the last tool | Team decision | Yes | – | – |
| [Q-05](#q-05--budget-and-runtime-per-run) | Budget and runtime per run | Team decision | Yes | – | A1, A2 |
| [Q-06](#q-06--region) | Region | Team decision | Yes | – | A3 |
| [Q-07](#q-07--cloud-provider) | Cloud provider | Team decision | Yes | – | A10 |
| [Q-08](#q-08--masked-text-abroad) | Masked text abroad (only if Sri Lanka is required) | Team decision | No | – | – |
| [Q-09](#q-09--unit-of-an-entry) | Unit of an entry | Team decision | No | – | A4 |
| [Q-10](#q-10--normal-off-peak-volume) | Normal off-peak volume | Team decision | No | – | A7 |
| [Q-11](#q-11--retention) | Retention | Team decision | No | – | – |
| [I-01](#i-01--the-earlier-infrastructure-slide) | Earlier infrastructure slide (instructors) | Team decision | Yes | – | A9 |
| [I-02](#i-02--demo-cloud-account) | Demo cloud account (instructors) | Team decision | Yes | – | A10 |

## Questions to the client

### Q-01 · Risk framework

**Question:** Which ISA 240 criteria does the firm weight most heavily, how does materiality set the thresholds, and are there entries that must always be selected?

**Why it matters:** it sets the criterion weights, and therefore the ranking.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** yes, as "please confirm"

**Team decision:** the brief's five criteria, weighted equally (1.0 each) and defined as in the sample working paper:
- C1: account used three times or fewer in the year, or in the suspense/clearing class;
- C2: user posted fewer than 10 journals in the year, or is a system or IT user posting manually;
- C3: a multiple of LKR 50,000 and at least LKR 1,000,000;
- C4: last 5 days of the year, outside 08:00–18:00 Sri Lanka time, or on a weekend or public holiday;
- C5: narration empty, shorter than 12 characters, or only generic words.

The score is multiplied by 1.5 at or above performance materiality. No must-include rules and no random picks for now.

**Affects:** A6, ADR-001, the risk-framework spec.

### Q-02 · What "good enough" means

**Question:** How many false alarms will a team tolerate among the 300 selected lines, and what detection rate would the firm accept?

**Why it matters:** it sets the pass thresholds for the locked evaluation.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** yes, as "please confirm"

**Team decision:** targets the team sets before the locked evaluation ([evaluation plan](evaluation-plan.md)):
- every seeded anomaly type appears at least once in the top 300;
- at least 80% of seeded scenarios have a line in the top 300 (scenario recall@300 ≥ 80%);
- at most two false alarms for every real anomaly in the top 300 (precision@300 ≥ 33%).

**Affects:** A8, the evaluation spec.

### Q-03 · Working-paper format

**Question:** What must the working paper contain for an engagement reviewer to sign it off?

**Why it matters:** it defines the golden path's output and how it is accepted.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** yes, as "please confirm"

**Team decision:** the team's 4-page sample of 29 Sep 2026:
- a header and sign-off block (preparer, reviewer, manager or partner);
- the population and trial-balance tie-out;
- the criteria applied;
- the selected lines, with columns for the team's testing;
- one page per case;
- the tickmark legend;
- a conclusion box for the team;
- the run provenance.

It is delivered as a PDF with CSV appendices.

**Affects:** A5, ADR-002, the working-paper spec.

### Q-04 · Current practice and the last tool

**Question:** What do engagement teams do today, and why do they distrust the last tool the firm bought?

**Why it matters:** it shows which failures would make the client reject Tickmark.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** yes, as "please confirm"

**Team decision:** teams pick about 300 lines with a few crude filters and professional instinct, which takes 2–3 days (brief §§1–2). The team's view is that the last tool lost their trust because its flags were unexplained or far too many. Tickmark's answer is that every flag names its criterion and source record, the list is capped at the team's review capacity, and disagreements are shown rather than hidden.

**Affects:** positioning (scope §9), and the evaluation's awkward and hostile cases.

### Q-05 · Budget and runtime per run

**Question:** Does the USD 40 ceiling cover all 3–4 runs of an engagement, or each run? Does the 4-hour limit apply to each run?

**Why it matters:** it decides how much model work each run can afford.

**Raised:** 24 Sep 2026 · **Status:** Team decision (29 Sep 2026) · **Send:** yes, as "please confirm"

**Team decision:** each run is capped at USD 30–40, and the design targets about USD 20 worst case per run. An engagement's 3–4 runs then stay under USD 40 even if the ceiling is per engagement, because re-runs reuse stored agent outputs. The 4-hour limit applies to each run.

**Affects:** A1, A2, ADR-014, ADR-018.

### Q-06 · Region

**Question:** Must ledgers stay in Sri Lanka, or may they be stored and processed in another country the firm approves? We propose India, across two cloud regions (Mumbai and Hyderabad), covering storage, processing, the AI model, backups and logs.

**Why it matters:** no major cloud provider has a region in Sri Lanka, so multi-region hosting with a managed AI model is possible only outside it. "Sri Lanka only" means a self-hosted model at a single site.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026; replaces Japan, chosen earlier the same day, and the 29 Sep answer "Sri Lanka only") · **Send:** yes, as "please confirm"

**Team decision:** India is acceptable. Everything that touches client ledgers stays in India: storage, processing, the AI model, backups, and logs or traces that contain ledger data. Nothing is processed in any other country.

**Affects:** A3, ADR-015, ADR-016, ADR-018, ADR-019.

### Q-07 · Cloud provider

**Question:** Is a public cloud provider, Amazon Web Services, acceptable for client ledgers, if they are encrypted at rest and in transit and each engagement's data is reachable only by its own team?

**Why it matters:** it decides whether Tickmark can use managed cloud services and a managed AI model, or must run on servers the firm controls.

**Raised:** 29 Sep 2026 · **Status:** Team decision (30 Sep 2026; replaces the 29 Sep answer "a Sri Lankan data centre") · **Send:** yes, as "please confirm"

**Team decision:** yes, AWS is acceptable on those terms.

**Affects:** A10, ADR-015, ADR-016, ADR-018.

### Q-08 · Masked text abroad

**Question:** Only if ledgers must stay in Sri Lanka (Q-06): would the firm allow masked text, with no names, amounts or account numbers, to be processed by an AI model in India?

**Why it matters:** a yes keeps a managed model while the ledgers stay in Sri Lanka. A no means self-hosting the model in Sri Lanka.

**Raised:** 29 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** no; needed only if the client rejects India (Q-06)

**Team decision:** not needed while the client accepts India (Q-06). If it rejects India, the answer is no, and we fall back to the self-hosted Sri Lankan design (ADR-015).

**Affects:** ADR-015, ADR-018.

### Q-09 · Unit of an entry

**Question:** Is an "entry" a journal line or a whole journal?

**Why it matters:** it changes the ledger schema, the simulator and every metric.

**Raised:** 24 Sep 2026 · **Status:** Team decision (29 Sep 2026) · **Send:** no; the team defines the ledger format

**Team decision:** one journal line. Lines of the same journal share a case.

**Affects:** A4, ADR-003, ADR-012, the ledger-schema spec.

### Q-10 · Normal off-peak volume

**Question:** How many engagements does the firm run in a normal month outside January–March?

**Why it matters:** it is needed for the normal monthly cost and the break-even point.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** no; the cost model covers 1, 4 and 12 engagements a month

**Team decision:** unknown. The cost model shows 1, 4 and 12 engagements a month outside January–March.

**Affects:** A7, ADR-014, ADR-018.

### Q-11 · Retention

**Question:** How long must ledgers, run results and working papers be kept?

**Why it matters:** it sets storage size and cost, and when data must be deleted.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** no; the team follows the usual retention minimum for audit documentation

**Team decision:** working papers and run results are kept for 5 years, the common minimum for audit documentation. Uploaded ledgers are deleted 90 days after the engagement is signed off. Deletion also covers the copies and backups in the standby region.

**Affects:** ADR-016, the cost model.

## Questions to the instructors

### I-01 · The earlier infrastructure slide

**Question:** Does the earlier four-criterion slide (multi-cloud, autoscaling under 10× load, observability and CI/CD, zero-downtime with LLM fallback) still apply alongside the handbook?

**Why it matters:** the design is already multi-region within India (ADR-015). If the slide applies, multi-cloud and zero-downtime releases would also be required.

**Raised:** 24 Sep 2026 · **Status:** Team decision (30 Sep 2026) · **Send:** yes, to the instructors

**Team decision:** no. The handbook's six weighted areas govern assessment, and multi-region is the team's own choice.

**Affects:** A9, ADR-015, ADR-020.

### I-02 · Demo cloud account

**Question:** May the demo run in the team's own AWS account, in Mumbai and Hyderabad, using synthetic ledgers only, and does the course provide cloud credits?

**Why it matters:** the demo needs Claude on Bedrock in India plus a standby region, and the cost model must keep credits apart from real prices.

**Raised:** 30 Sep 2026 · **Status:** Team decision (30 Sep 2026; replaces the 29 Sep question about a team member's GPU machine) · **Send:** yes, to the instructors

**Team decision:** yes to the team account. The team assumes no credits, so the cost model uses list prices.

**Affects:** A10, ADR-018.

## Change record

When a reply changes a team decision, add a row here and update the question above.

| Date | ID | Team decision | Reply | What we updated | Effect on golden path, evidence, cost and priorities |
|---|---|---|---|---|---|
| – | – | – | – | – | No replies yet |
