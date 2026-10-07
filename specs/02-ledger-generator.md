# Spec 02 · Ledger generator

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned, and not the owner of spec 03 (GR-32) · **Prefix:** GEN · **ADRs:** ADR-003, ADR-004 · **Ground rules:** GR-04, GR-05, GR-07, GR-21, GR-25, GR-26, GR-31, GR-32

The generator builds full-size synthetic ledgers for two fictional Sri Lankan companies. It plants fraud scenarios and innocent look-alikes in them, and writes the answers to a separate label folder. Its ledgers feed development, the locked evaluation and the load test. It writes them in the format of [spec 01](01-ledger-format.md), so they are submitted like any other ledger.

## 1. Purpose

Real client ledgers are out of scope. The brief asks for a full-size synthetic ledger with seeded anomalies, so that precision and recall can be measured, and it does not accept results from a few thousand rows (brief §7, ADR-004). The generator supplies those ledgers, keeps the known answers apart from them, and rebuilds any ledger exactly from its recorded seed. It serves layer 1 of the [evaluation plan](../docs/evaluation-plan.md) and the load test.

## 2. Scope

**In scope:**
- Two business profiles: retail and wholesale trading.
- Normal activity, planted fraud scenarios and innocent look-alikes.
- Labels, the generation manifest, and the four ledger sets: fixture, development, locked test and load.
- Rebuilding a ledger exactly from its seed.

**Out of scope:**
- The ledger format and its checks (spec 01).
- The criteria and their thresholds (spec 03). The generator never reads them.
- Malformed inputs for spec 01's tests. Those are small hand-written fixtures, not generator output.
- Converting public datasets, and scoring results against the labels (spec 10).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| GEN-01 | Tickmark's generator shall write each ledger in spec 01's format, with every optional column present, together with a label folder and a generation manifest (§4.6). | ADR-004 |
| GEN-02 | The generator shall support two business profiles, retail and wholesale trading, each defined in a versioned profile file (§4.1, §4.2). | ADR-004 |
| GEN-03 | The generator shall produce exactly the requested number of journal lines, 400,000 by default, planted lines included. | Brief §7 |
| GEN-04 | When a generated ledger is submitted, Tickmark shall accept every row and raise only the warnings that scenarios plant on purpose. | Spec 01 |
| GEN-05 | The generator shall create amounts as integer minor units and balance every journal in integer arithmetic. | GR-05 |
| GEN-06 | When the simulator version, profile version, catalogue version, seed and line count are the same, the generator shall write byte-identical files, whatever the machine or the number of workers. | GR-04 |
| GEN-07 | The generator shall take all randomness from the seed, with a separate stream for each part of the work, and shall not read the clock or the environment. | GR-07 |
| GEN-08 | The generator shall produce a 400,000-line ledger within 10 minutes and 4 GB of memory on a developer laptop. | Evaluation plan |
| GEN-09 | The generator shall produce each profile's normal activity from its transaction mix (§4.2), so that the ledger meets the realism checks in §4.3. | Evaluation plan |
| GEN-10 | The generator shall write narrations in English, the way accounting staff write them, from phrase banks for each transaction type, with variation, including short narrations that are legitimate. | Evaluation plan |
| GEN-11 | The generator shall plant scenarios from the catalogue in §4.4, built from known fraud patterns, and shall never read spec 03's criteria or any run settings. | Evaluation plan |
| GEN-12 | Each 400,000-line ledger shall hold at least 60 scenarios and at least 6 of each type, with 120 to 300 planted entries in total and 25% to 40% of scenarios in their weak variant. | Evaluation plan, A8 |
| GEN-13 | Each 400,000-line ledger shall hold at least 1,000 innocent look-alikes (§4.5), with at least 150 for each warning sign, C1 to C5. | Evaluation plan |
| GEN-14 | The generator shall write labels only to the label folder (§4.6), recording each planted entry's scenario, each scenario's type, variant and designed warning signs, and the sign each look-alike mimics. | GR-25 |
| GEN-15 | The generator shall leave no trace of the labels in the submission files: no label column, no scenario ID, and no hint in any value. Entry and journal IDs shall follow posting order, so that planted entries cannot be spotted by their ID or their place in the file. | GR-25 |
| GEN-16 | The generator shall use different seeds for the development, locked test and load sets, and locked test ledgers shall also use scenario variants and parameters held back from development. | GR-31 |
| GEN-17 | The seeding owner shall seal the locked test seeds and held-back variants in one private file outside the repository, and commit only its SHA-256 hash before the rules are frozen (§4.8). | GR-31, GR-32 |
| GEN-18 | The team shall never commit generated ledgers or labels. The repository holds the generator, the profiles, the scenario catalogue, the holiday input files and the seeds. | GR-26 |
| GEN-19 | The generator's profiles shall be fictional, with invented entity, people, customer and supplier names, and no real company's records. Where a profile draws on a real company's statistics, the team shall hold that company's written permission and use aggregates only. | GR-21 |
| GEN-20 | The generator shall read each fiscal year's public holidays, including Poya days, from an input file, and copy them unchanged into `engagement.json`. An empty list is allowed. | Spec 01 |
| GEN-21 | The seeding owner shall generate the locked test ledgers only after the rule bundle is frozen and its commit tagged in Git, and shall commit the sealed file once the locked evaluation has run, so that the locked ledgers can be rebuilt (§4.8). | GR-31 |

## 4. Data shapes

### 4.1 Profile file

Each profile is one versioned file in the repository.

| Part | What it holds |
|---|---|
| Identity | `profile_id` and `version`, such as `retail` and `1.0` |
| Entity | A fictional name, the currency (LKR, 2 minor digits), the timezone (`Asia/Colombo`) and the fiscal year |
| Calendar | Working days and hours, the holiday input file to use, and the close calendar. To start with, each month closes on the 5th working day after it ends, and the year closes on the 20th working day after it ends. |
| Accounts | The chart of accounts, using spec 01's classes, and how often each account is used |
| Users | People with their roles and posting activity (regular or occasional), and system accounts |
| Counterparties | Fictional customers and suppliers, named in narrations |
| Transaction types | For each type: its share of lines, the lines per journal, the accounts it uses, its amount distribution, its source, who posts it, when it is posted, and its narration phrases |
| Settings for spec 03 | The approval limit and materiality, as decimal text. They go into the generation manifest for the run settings. |

### 4.2 The two profiles

**Retail:** Tickmark Demo Retail (Pvt) Ltd, a fictional chain of 30 supermarkets. Fiscal year 1 April 2025 to 31 March 2026, about 220 accounts, about 40 people and 4 system accounts.

| Transaction type | Share of lines | Source | Posted by |
|---|---|---|---|
| Daily store sales from the tills, by department | 30% | `interface` | System account |
| Card and cash settlements, and bank deposits | 12% | `interface` or `manual` | System account, cashiering staff |
| Purchase invoices for stock | 18% | `manual` | Payables clerks |
| Supplier payments | 10% | `manual` | Payables clerks |
| Stock movements and shrinkage adjustments | 8% | `automated` | System account |
| Payroll | 5% | `interface` | Payroll system account |
| Rent, utilities, marketing and other expenses | 7% | `manual` | Accounting staff |
| Month-end accruals and their reversals | 5% | `manual` | Senior accountants |
| Depreciation and other recurring journals | 3% | `automated` | System account |
| Year-end and other adjustments | 2% | `manual` | Finance manager, senior accountants |

**Wholesale trading:** Tickmark Demo Wholesale (Pvt) Ltd, a fictional importer that sells garden tools and equipment to hardware shops, like the company in the Gronewald et al. dataset. Fiscal year 1 January to 31 December 2025, about 180 accounts, about 30 people and 3 system accounts.

| Transaction type | Share of lines | Source | Posted by |
|---|---|---|---|
| Sales invoices to business customers, on credit | 28% | `interface` | System account (order system) |
| Customer receipts | 14% | `manual` | Receivables clerks |
| Purchase invoices for stock | 20% | `manual` | Payables clerks |
| Supplier payments | 12% | `manual` | Payables clerks |
| Freight, warehousing and import costs | 6% | `manual` | Accounting staff |
| Stock movements and write-downs | 6% | `automated` | System account |
| Payroll | 4% | `interface` | Payroll system account |
| Other operating expenses | 4% | `manual` | Accounting staff |
| Month-end accruals and their reversals | 4% | `manual` | Senior accountants |
| Depreciation and other recurring journals | 1% | `automated` | System account |
| Year-end and other adjustments | 1% | `manual` | Finance manager, senior accountants |

### 4.3 Realism checks

Each check counts normal activity and look-alikes, not planted scenarios. The domain advisor reviews the targets.

| Check | Target |
|---|---|
| Manual journals by people, entered on working days (not weekends or listed holidays) within working hours | At least 90% |
| System batch journals entered between 00:00 and 05:00 | At least 95% |
| Lines posted in each month's last 3 working days | 15% to 30% of that month's lines |
| Lines posted in the fiscal year's last 5 days | 2.5% to 6% of the year's lines |
| Month-end accruals reversed on the next period's first working day | All of them |
| Accounts legitimately used 3 times or fewer in the year | At least 10 |
| People who legitimately post fewer than 10 journals in the year | At least 3 |
| Lines per journal | 2 to 200 |
| Each transaction type's share of lines | Within 1 percentage point of its profile's mix |

### 4.4 Scenario catalogue

Each scenario is made of one or more balanced journals that look normal in every other way. A strong variant shows several warning signs, and a weak variant shows only one, so that recall is not flattered.

| Type | Fraud pattern | Journals | Strong variant shows | Weak variant shows |
|---|---|---|---|---|
| S01 | Sales recorded in the last days of the year with no real customer order, and reversed after year end, outside this ledger. The label records the planned reversal. | 1–2 | C4, C3, C5 | C4 |
| S02 | One payment split into 2 to 5 payments to the same supplier within a few days, each just under the approval limit | 2–5 | C5, C4 | C5 |
| S03 | Money parked in a suspense or clearing account, then cleared to expenses or cash weeks later | 2–3 | C1, C3, C5 | C1 |
| S04 | A journal keyed by someone who rarely posts, by a system account used by hand, or by a user missing from the user list | 1–2 | C2, C4, C5 | C2 |
| S05 | An entry dated in a closed period, or keyed after the year was closed, with little or no description | 1–2 | C4, C5 | C4 |
| S06 | A round sum posted to a rarely used account | 1–2 | C1, C3 | C3 |
| S07 | An entry joining accounts that do not normally go together, such as revenue credited against a fixed asset | 1 | C1, C4, C5 | C1 |

- Round sums are chosen the way people pick them, such as whole hundreds of thousands or millions of rupees, not from spec 03's thresholds.
- A scenario may combine patterns, for example S01 keyed by an occasional poster.
- The seeding owner holds back at least one variant of each type for the locked test set.
- At least 120 planted entries keep the 33% precision pass mark reachable, because precision@300 at 33% needs at least 99 planted entries in the top 300 (evaluation plan, A8).

### 4.5 Innocent look-alikes

| Sign | Legitimate entries that show it |
|---|---|
| C1 | Accounts used a few times a year, such as donations, insurance claims and asset disposals; suspense entries cleared within days |
| C2 | Year-end adjustments by the finance manager; a relief clerk covering leave |
| C3 | Rent, loan repayments, capital injections and fixed fees in round sums |
| C4 | Year-end accruals and closing entries; scheduled night-time batches; month-end overtime |
| C5 | Short but genuine narrations on routine postings, such as "Bank chgs Mar"; system batches described by a code only |

### 4.6 Output

Each ledger is written to its own folder outside the repository:

```text
<ledger_id>/
  submission/      entries.csv, accounts.csv, users.csv, engagement.json  (spec 01)
  labels/          scenarios.csv, entries.csv
  generation.json
```

Only the `submission/` folder is ever uploaded to Tickmark.

**`generation.json`:**

| Field | Meaning |
|---|---|
| `ledger_id` | Such as `retail-dev-01` |
| `set` | `fixture`, `development`, `locked` or `load` |
| `profile_id`, `profile_version`, `catalogue_version` | What was generated |
| `simulator_version` | The generator's package version and Git commit |
| `seed` | The seed used |
| `lines`, `journals` | Counts generated |
| `scenarios`, `planted_entries`, `lookalikes` | Counts planted |
| `control_totals` | Total debits and credits in minor units, which must equal spec 01's validation report |
| `files` | The SHA-256 hash of every file written |
| `profile_settings` | The approval limit and materiality, as decimal text, for spec 03's run settings |

**`labels/scenarios.csv`:** `scenario_id`, `type` (`S01` to `S07`), `variant` (`strong`, `weak`, or a held-back variant's ID), `designed_signs` (such as `C1;C3`), `journal_ids` (separated by `;`), `note`.

**`labels/entries.csv`:** `entry_id`, `kind` (`fraud` or `lookalike`), `scenario_id` (fraud only), `mimics` (look-alikes only, `C1` to `C5`). Entries not listed are normal.

### 4.7 Ledger sets

| Set | Ledgers | Seeds | Used for |
|---|---|---|---|
| Fixture | 1 per profile, 2,000 lines | In the repository | Unit tests, the CI determinism test (ADR-019) and FMT-T14 |
| Development | 2 per profile, 400,000 lines | In the repository | Building and tuning the rules and agents |
| Locked test | 2 per profile, 400,000 lines | Sealed: only their hash is committed until the locked evaluation has run (§4.8) | The evaluation report's main results |
| Load | 6 per profile, 400,000 lines | In the repository | The load test's 12 concurrent runs |

### 4.8 Sealing the locked seeds

The locked test only means something if nobody tunes the rules to it. These steps keep its seeds hidden until the rules are frozen, and let the assessors check afterwards that nothing changed.

1. **Seal:** the seeding owner writes one private file, `locked-seeds.json`, holding the locked seeds, the held-back variants' parameters, and a random value of 32 bytes. The random value stops anyone working out the seeds from the hash.
2. **Commit the hash:** the owner commits only the file's SHA-256 hash, as `locked-seeds.json.sha256`, next to the generator. This happens before the rules are frozen.
3. **Keep it safe:** the owner keeps the file in their password manager, and gives a backup copy to a teammate who does not own spec 03.
4. **Freeze:** the team freezes the rule bundle and tags that commit in Git.
5. **Generate and run:** only then does the seeding owner generate the locked ledgers, and the team runs the locked evaluation.
6. **Open:** the owner commits `locked-seeds.json`, and CI checks that its hash matches the one committed in step 2.

The Git history then shows the order: hash, freeze, evaluation, file. These steps rule out AWS Secrets Manager, where any teammate with admin rights could read the file, and GitHub Actions secrets, which anyone who edits a workflow could print.

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| GEN-T01 | The retail profile, a development seed and 400,000 lines | The ledger is generated, then submitted to spec 01 | It has exactly 400,000 lines. Every row is accepted, the only warnings are on planted unknown-user entries, and generation took under 10 minutes and 4 GB. *Ordinary* | GEN-01, GEN-03, GEN-04, GEN-05, GEN-08 |
| GEN-T02 | The wholesale profile, a development seed and 400,000 lines | The ledger is generated, then submitted to spec 01 | The same results as GEN-T01. *Ordinary* | GEN-01, GEN-02, GEN-03, GEN-04, GEN-05 |
| GEN-T03 | The same inputs | The ledger is generated twice, on a laptop and in CI, with 1 and then 4 workers | The files are byte-identical, and spec 01 gives the same snapshot hash. | GEN-06 |
| GEN-T04 | The same inputs, with the system clock moved a year ahead | The ledger is generated | The files are byte-identical to GEN-T03's. | GEN-07 |
| GEN-T05 | Each development ledger | The realism checks run | Every check meets its target (§4.3). | GEN-09 |
| GEN-T06 | Each development ledger | Its narrations are compared with the phrase banks | Every transaction type uses at least 5 different phrasings, and some legitimate narrations have fewer than 3 words. | GEN-10 |
| GEN-T07 | Each development ledger's labels | The scenarios are counted | At least 60 scenarios and 6 of each type, 120 to 300 planted entries, and 25% to 40% weak variants. | GEN-11, GEN-12 |
| GEN-T08 | Each development ledger's labels | The look-alikes are counted | At least 1,000, with at least 150 for each sign. | GEN-13 |
| GEN-T09 | A generated ledger | Its labels are checked against its submission files | Every labelled entry exists, and every planted journal is listed. The submission files hold no scenario ID, label word or label column, and planted IDs follow posting order like all others. | GEN-14, GEN-15 |
| GEN-T10 | The manifests of the development, locked and load sets | They are compared | No seed is shared, and no held-back variant appears in a development ledger's labels. | GEN-16 |
| GEN-T11 | The repository | CI searches it | It holds no generated ledger or label file, and no locked seed before the locked evaluation. | GEN-17, GEN-18 |
| GEN-T12 | The fixture set | It is generated in CI | Two 2,000-line ledgers, identical on every run, which the determinism test and FMT-T14 use. | GEN-03, GEN-06 |
| GEN-T13 | A holiday input file, then an empty one, and the profiles | Ledgers are generated, and the profiles are reviewed in the pull request | `engagement.json` lists exactly the input's holidays, and the empty list works. Every name is invented and matches no company found in a web search. | GEN-19, GEN-20 |
| GEN-T14 | A generated ledger's `generation.json` | It is compared with spec 01's validation report | The line counts and control totals are equal. | GEN-01 |
| GEN-T15 | The committed hash, the freeze tag and the sealed file | CI runs after the sealed file is committed | The file's SHA-256 hash equals the committed one. The Git history shows the hash committed before the freeze tag, and the file committed after the locked evaluation's results. | GEN-17, GEN-21 |

## 6. Open questions

None.

**Answered on 7 Oct 2026 (team decisions):**
1. The holidays are an input file for each fiscal year, supplied when ledgers are generated. It may be empty, in which case only weekends count as non-working days (GEN-20).
2. Yes: April 2025 to March 2026 for retail, and calendar 2025 for wholesale.
3. English only (GEN-10).
4. Yes: at least 60 scenarios, with up to 300 planted entries in a 400,000-line ledger, is realistic.
5. Sealed: the locked seeds stay in a private file whose hash is committed before the rules are frozen, and whose contents are committed after the locked evaluation (§4.8).

---

*Created 2026-10-07.*
