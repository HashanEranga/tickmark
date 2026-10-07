# Spec 10 · Evaluation

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** EVL · **ADRs:** ADR-004, ADR-007, ADR-014, ADR-016, ADR-018 · **Ground rules:** GR-04, GR-21, GR-25, GR-26, GR-27, GR-31 to GR-33

The evaluation harness measures what the handbook's evidence deliverables need: detection quality against the planted scenarios, agent quality, reproducibility, the named cases, load and cost. It runs ledgers through the Run API exactly as a user would, reads the labels only afterwards, and writes the numbers that go into the evaluation report, the load-test report and the cost model. This spec turns the [evaluation plan](../docs/evaluation-plan.md) into requirements.

## 1. Purpose

The handbook fails a demo that has no evaluation, scale test or cost arithmetic, and pays 20% for evidence and 15% for commercial thinking (handbook §3). The brief asks for precision and recall against seeded anomalies, with an honest account of what is missed (brief §11). This spec makes those measurements repeatable and keeps them honest: the rules are frozen before the locked test, labels never reach scoring, and failures are reported alongside passes.

## 2. Scope

**In scope:**
- The harness, the detection metrics and the baselines.
- The pass thresholds and the locked evaluation.
- The agent metrics and the comparison of routing modes.
- The named cases and reproducibility checks.
- The dataset converters, the load test, the cost data and the report's structure.

**Out of scope:**
- Generating ledgers (spec 02).
- The cost model's pricing scenarios, which the cost-model deliverable builds from this spec's data.

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| EVL-01 | The harness shall run each evaluation ledger through the Run API, and record its ledger ID, generation manifest, run ID, manifest hash and decision hash. | Scope §5 |
| EVL-02 | The harness shall read a ledger's labels only after the run's scoring record exists, and shall never upload labels to Tickmark. | GR-25 |
| EVL-03 | The harness shall compute, for each layer and profile, the metrics in §4.1 at k = 300, using the actual list size, labelled as the cut-off, when fewer entries are flagged, and reporting metrics with a zero denominator as undefined. | Evaluation plan |
| EVL-04 | The harness shall compute the same metrics for the three baselines in §4.1, on the same ledgers and review budget. | Scope §5 |
| EVL-05 | The harness shall judge layer 1's locked test set against the pass thresholds in §4.2, and report each as passed or failed. | Evaluation plan, A8 |
| EVL-06 | The team shall run the locked evaluation as in §4.3: rules frozen and tagged first, locked ledgers generated after that from the sealed seeds, and each locked ledger run once. Any later change starts a new evaluation version. | GR-31, spec 02 |
| EVL-07 | The harness shall report the agent metrics in §4.4 for layers 1 and 2. | Scope §5 |
| EVL-08 | The harness shall run the four routing modes (spec 05) on the same cases and allowance, and recommend code-only routing if the router's added-value rate is below 10%. | ADR-007 |
| EVL-09 | The harness shall collect every acceptance test marked ordinary, awkward, hostile or recovery from specs 01 to 09 into the named-case table, with each one's ID, expected and actual result. It shall report passed cases out of executed cases, and show failed, blocked and unexecuted cases separately. | Scope §5, GR-33 |
| EVL-10 | The harness shall repeat each development run with shuffled input and a different worker count, and confirm that the decision hash is identical. | GR-04 |
| EVL-11 | Where the Gronewald et al. data is available, its converter shall map it to spec 01's format as in §4.5, never inventing a value, and write its labels to a separate folder. | Evaluation plan, GR-25 |
| EVL-12 | The Schreyer converter shall map each row to a two-line journal as in §4.5. The harness shall leave the offset lines out of every metric and every k. | Evaluation plan |
| EVL-13 | Every converter's output shall pass spec 01's validation, live outside the repository, and be recorded with its source file's checksum. | GR-26 |
| EVL-14 | The team shall run the load test in §4.6, and record its workload, environment and method with the results. | Scope §6, GR-27 |
| EVL-15 | The harness shall record each run's measured model tokens by role and model, its compute time and storage, and its worst-case model cost, for the cost model. | Scope §6, ADR-014 |
| EVL-16 | The harness shall write the evaluation report in the structure of §4.7, including known misses and their causes, and claiming neither ISA 240 or SLAuS 240 compliance nor real-world detection. | GR-33 |
| EVL-17 | The harness shall reproduce every number in the report from the stored run outputs and labels, under a versioned evaluation ID. | GR-04 |

## 4. Data shapes

### 4.1 Metrics and baselines

| Metric | Meaning |
|---|---|
| Entry recall@k | Planted entries in the top k, divided by all planted entries |
| Scenario recall@k | Scenarios with at least one entry in the top k, divided by all scenarios |
| Precision@k | Planted entries in the top k, divided by k |
| First-hit rank | The median rank of each scenario's first entry in the ranking; scenarios never found are listed separately |
| Recall by type and criterion | Scenario recall@k for each fraud type (S01 to S07), and for each designed warning sign |
| Look-alike rate | Innocent look-alikes in the top k, divided by k, for each sign they mimic |

| Baseline | Ranking |
|---|---|
| Random | A seeded random order of all accepted entries |
| Largest first | All accepted entries by amount, largest first |
| Rules without ranking | The flagged entries in posting order, without scores |

### 4.2 Pass thresholds (layer 1, locked set)

Set by the team before the locked evaluation (evaluation plan):
- every seeded fraud type appears at least once in the top 300;
- scenario recall@300 is at least 80%;
- precision@300 is at least 33%.

### 4.3 Locked evaluation

1. The spec 02 owner has committed the sealed seeds' hash (spec 02 §4.8).
2. The team freezes the rule bundle, the agent profile and the prompts, and tags the commit.
3. The seeding owner generates the locked ledgers from the sealed seeds.
4. The harness runs each locked ledger once, then scores it against its labels.
5. The seeding owner commits the sealed file, and CI checks it against the hash.
6. Any later change to rules, settings, prompts or models creates a new evaluation version, reported separately.

### 4.4 Agent metrics

| Metric | Meaning |
|---|---|
| First-time pass rate | Drafts that pass the verifier at once, out of all drafts |
| Repair success rate | Repaired drafts that pass, out of all repairs |
| Template rate by reason | Entries with each template outcome (spec 06), out of all selected entries |
| Router added-value rate | Cases where a specialist the router added, beyond the required ones, raised a concern citing evidence that no required specialist cited, out of cases where the router added one |
| Narration agreement | Seeded weak-narration entries that the narration specialist assessed as a concern, out of those it saw |
| Human review score | The domain advisor's scores for 30 sampled justifications per profile, from 1 to 3 on each of: accurate, cites its evidence, covers its criteria, draws no conclusion, useful to a reviewer |

The routing comparison reports every metric above, together with cost and runtime, for each of the four routing modes.

### 4.5 Converters

**Gronewald et al. (layer 2):** posting lines become entries, and transactions become journals. The date fields, the account, amount, debit or credit flag, user and posting text map to their spec 01 columns. A mapping file assigns SKR03 account ranges to spec 01 classes, which the audit adviser reviews; accounts it cannot place are `unclassified`. A field the data lacks is left out as a column, never filled in. Labels go to a separate folder.

**Schreyer et al. (layer 3):** each row becomes a two-line journal:
- the original line, with `BELNR` and the row number as its IDs, `HKONT` as its account, `DMBTR` as its amount, and side `D`;
- an offset line for the same amount, on a single offset account of class `unclassified`, with side `C`.

The engagement's minor digits are the most decimal places the amounts use, up to 3. Amounts that need more are left as they are, so spec 01 rejects them, and the report counts them. Only C1 and C3 run.

### 4.6 Load test

| Aspect | Plan |
|---|---|
| Workload | 12 load ledgers of 400,000 entries, 6 per profile, with runs started within 5 minutes of each other |
| Environment | The cloud setup of spec 09, with the granted Bedrock quotas, the worker task sizes and the concurrency limit recorded |
| Measured | Each run's duration against the 4-hour limit, its queue wait, the API's 95th and 99th percentile response times, its error rate, the number of throttled model calls, and the cost per run |
| Pass | All 12 runs finish within 4 hours, none waits more than 10 minutes to start, under 1% of API requests fail, and every engagement stays under USD 40 |

### 4.7 Evaluation report

1. Summary, with pass or fail against each threshold.
2. The data: profiles, ledgers, seeds and generation manifests.
3. Detection results for each layer and profile, against the baselines.
4. Agent results and the routing comparison.
5. Reproducibility results.
6. Named cases, with failed, blocked and unexecuted cases shown separately.
7. Known misses and their causes, plus the limitations, such as synthetic data and consistent ending numbers not being tested.
8. The evaluation versions and what changed between them.

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| EVL-T01 | A development ledger and its labels | The harness runs it | The run goes through the Run API, the IDs and hashes are recorded, and the labels are read only after the scoring record exists. *Ordinary* | EVL-01, EVL-02 |
| EVL-T02 | A hand-built ranking with known planted entries | The metrics are computed | Every metric in §4.1 matches the hand-computed value, including an undefined one with a zero denominator. | EVL-03 |
| EVL-T03 | A fixture with 40 flagged entries | The metrics are computed | k is 40 and labelled as the cut-off. *Awkward: small population* | EVL-03 |
| EVL-T04 | A development ledger | The baselines run | All three are computed on the same ledger and budget, and the random baseline repeats with its seed. | EVL-04 |
| EVL-T05 | A locked-set result just below 80% scenario recall | It is judged | The threshold is reported as failed, with the number. | EVL-05 |
| EVL-T06 | The Git history after a locked evaluation | It is checked | The sealed hash comes before the freeze tag, the locked runs after it, and the sealed file after the runs. | EVL-06 |
| EVL-T07 | A development run | The agent metrics are computed | Every metric in §4.4 is reported, and the human review sample has 30 justifications per profile. | EVL-07 |
| EVL-T08 | The four routing modes on the same cases | They are compared | Each mode reports its metrics, cost and runtime, and an added-value rate below 10% produces the code-only recommendation. | EVL-08 |
| EVL-T09 | The specs' acceptance tests with their markings | The named-case table is built | Every marked test appears with its result, and the passed, failed, blocked and unexecuted counts add up. | EVL-09 |
| EVL-T10 | Each development ledger | It is rerun with shuffled rows and a different worker count | The decision hash is identical. | EVL-10 |
| EVL-T11 | A small sample in the Gronewald format | It is converted | It passes spec 01, its labels are in a separate folder, and a field missing from the data is an absent column. | EVL-11, EVL-13 |
| EVL-T12 | The Schreyer file | It is converted and run | Every row becomes a balanced two-line journal, offset lines appear in no metric, only C1 and C3 run, and rows needing more decimal places are counted as rejected. | EVL-12, EVL-13 |
| EVL-T13 | The load test | It runs | Every pass condition in §4.6 is checked and recorded, with the environment. *Ordinary: peak load* | EVL-14 |
| EVL-T14 | A finished run | Its cost data is read | Tokens by role and model, compute time, storage and the worst-case cost are recorded, and the measured model cost is below the worst case. | EVL-15 |
| EVL-T15 | The stored outputs and labels of an evaluation version | The report is rebuilt | Every number matches the published report, in the structure of §4.7, including known misses. | EVL-16, EVL-17 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. The router keeps its place only if its added-value rate is at least 10%. The threshold is fixed now, before the locked evaluation (ADR-007).
2. Schreyer rows become two-line journals with an offset line, which keeps spec 01 unchanged. The offset lines are left out of every metric.
3. The engagement's minor digits for the Schreyer data follow the data's own decimal places, up to 3. Amounts needing more are rejected and counted rather than rounded.
4. The domain advisor reviews 30 justifications per profile, scored from 1 to 3 on five points.
5. The load test passes only if all 12 runs finish within 4 hours, none waits over 10 minutes, under 1% of API requests fail, and every engagement stays under USD 40.

---

*Created 2026-10-07.*
