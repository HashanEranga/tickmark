# Evaluation plan

How Tickmark's detection and agent quality will be measured. The main evaluation uses the team's own full-scale synthetic ledgers, as the brief requires (brief §7); two public datasets are run as an external check. The metrics come from [scope.md](scope.md) §5, the decision from [ADR-004](adr.md#adr-004--full-scale-synthetic-ledgers-checked-against-public-datasets), and the pass thresholds from the [client log](client-log.md) (Q-02). The results go into the evaluation report, one of the handbook's six deliverables.

## Three layers

| Layer | Data | What it shows | Pass mark |
|---|---|---|---|
| 1 · Main | Tickmark's synthetic ledgers: two business profiles, 400,000 journal lines each | Detection and agent quality at full scale, against known answers | Yes, the team's thresholds below |
| 2 · External | Gronewald et al. (2024) synthetic ledger of a trading company | How the frozen pipeline behaves on a ledger the team did not design; all five criteria can run | No, reported only |
| 3 · External | Schreyer et al. lab dataset: synthetic payments relabelled with SAP-style fields | How criteria C1 and C3 behave on a large labelled set the team did not design | No, reported only |

## Layer 1 · Tickmark's synthetic ledgers

**Business profiles.** Two fictional companies: a retailer, and a wholesale trading company chosen to match the company in the Gronewald et al. dataset (layer 2). Each profile sets its chart of accounts, users and roles, working hours, holidays (including Sri Lankan public and Poya holidays), close calendar, approval limits and materiality, all in the same ledger format. If a profile draws on a real company's ledger, the team needs that company's written permission and uses only aggregate statistics, never its rows, names or narrations.

**Normal entries.** Balanced journals; recurring entries such as payroll, rent and depreciation; accruals and their reversals; busier month and year ends; system users posting batches at night; and narrations written the way accounting staff write them.

**Seeded scenarios.** Drawn from known fraud patterns, not from Tickmark's criteria, for example:
- fictitious revenue near year end, reversed after it;
- payments split to stay under an approval limit;
- misuse of a suspense or clearing account;
- entries by users who rarely post, or by system accounts posting manually;
- backdated or post-closing entries with little or no description;
- round sums posted to rarely used accounts.

Each scenario records its fraud type and the ISA 240 warning signs (C1–C5) it shows. Some scenarios show only one weak sign, so recall is not flattered.

**Innocent look-alikes.** Legitimate entries that trip criteria, such as year-end accruals, a large round capital injection and scheduled night-time batch postings. Without them, precision means little.

**Separation.** The team member who seeds scenarios does not write the criteria. Labels are stored apart from the ledger, and neither scoring nor the agents can read them. Development ledgers are for tuning. The locked test ledgers use different seeds and scenarios, and run once, after the rules are frozen; any later tuning is a new evaluation version.

**Scale.** Every evaluation ledger has 400,000 journal lines (brief §7). The load test uses at least 12 ledgers. Each ledger is generated from a recorded seed and simulator version, so it can be rebuilt exactly.

## Layer 2 · Gronewald et al. (2024)

- **What it is:** a synthetic general ledger of a fictional garden-tools trading company, built at DFKI with the German standard chart of accounts (DATEV SKR03). It covers 2023, with 51,076 transactions in 127,835 posting lines and 573 labelled anomalies. Each line has document, posting and entry dates, the user, account, amount, debit or credit flag, tax rate and posting text, so all five criteria can run.
- **Get it:** the 2024 paper gives no download link. A later paper by Gronewald and Fettke (2026, SSRN) describes a two-year dataset built the same way; look there for the data and its licence. If no download is found within one week, drop this layer and say so in the report.
- **Convert:** map it into Tickmark's journal-line format and its accounts to canonical account classes, and write a business profile for the trading company. It matches Tickmark's wholesale trading profile, so the two can be compared directly.
- **Map labels:** the anomalies come from four journal entry tests: the largest amounts with a changed account and user, large cash payments by a changed user, late posting, and postings at unusual times or by unusual users. Tickmark's criteria cover all of them except late posting, which is reported as a miss by design.
- **Run:** the frozen rule bundle from the locked evaluation, with no tuning on this data.
- **Measure:**
  - All flagged entries as true and false positives. On this data the authors' journal entry tests flagged 10,592 entries to catch all 573 anomalies. A later study's rule-based tests caught all 50 anomalies in a 5,000-entry sample, but also flagged 942 innocent entries (AuditCopilot, Table 1).
  - The ranked list at k = 300, and at k equal to the number of labelled anomalies.
  - The agent team on the top of the list: verifier pass rate, template fallback rate and cost.

## Layer 3 · Schreyer et al. lab dataset

- **What it is:** the public dataset for the lab that accompanies Schreyer et al. (2017): 533,009 rows with 100 labelled anomalies, 70 global (rare values) and 30 local (rare combinations). It is not the paper's real SAP data, which was never published. The authors built it from PaySim, a synthetic mobile-payments dataset, and renamed its fields after SAP's. Each row has seven coded fields, including a general ledger account with 73 distinct values, and two amounts, but no users, dates or narration.
- **File:** `data/fraud_dataset_v2.csv` from [GitiHubi/deepAI](https://github.com/GitiHubi/deepAI): 27,044,366 bytes, SHA-256 `797acf17f64c4948126747028aca1692e55c091d64417280738bb6e37eba4a8f`, downloaded on 30 Sep 2026 to a data folder outside the repository.
- **Licence:** the repository is GPL-3.0, and PaySim, by Edgar Lopez-Rojas on Kaggle, is CC BY-SA 4.0. Use it for internal testing, credit both sources in the report, and do not redistribute it.
- **Run:** criteria C1 and C3 only; C2, C4 and C5 are reported as not testable. The agents do not run, because the rows have no users or narration to investigate.
- **Measure:** recall of the labelled anomalies in the top 300, and false alarms among the regular rows. Because the rows are synthetic payments renamed as postings, the results show how two criteria behave on data the team did not design, not on real business entries.
- **Priority:** below layer 2. If layer 2 is dropped, this becomes the only external check.

## Metrics

For each layer and business profile:
- entry recall@k and scenario recall@k, where a scenario counts as found when any of its entries is in the top k;
- precision@k;
- median rank of the first hit, with scenarios that are never found listed separately;
- recall by fraud type and by criterion;
- the same numbers for baselines on the same data and review budget: random selection, largest amounts first, and simple rules without ranking (scope §5).

k is 300. When fewer than 300 entries are flagged, the actual list size is used and labelled as the cutoff. Metrics whose denominator is zero are reported as undefined.

For the agent team (layers 1 and 2): the verifier pass rate, the template fallback rate, how often a specialist added by the router finds something the required ones missed, how often the narration specialist agrees with the seeded weak-narration cases, and a sampled review of justifications by the domain advisor. The routed team is compared with a fixed chain, code-only routing and a single agent on the same cases and budget (ADR-007).

## Pass thresholds (layer 1 only)

Set by the team before the locked evaluation, and sent to the client to confirm (client log Q-02):
- every seeded fraud type appears at least once in the top 300;
- scenario recall@300 is at least 80%;
- precision@300 is at least 33%, meaning at most two false alarms per real anomaly.

Layers 2 and 3 have no pass mark, because their labels were not designed around ISA 240's warning signs.

## Rules for the external datasets

- Check each dataset's licence before use. Do not commit the data; commit a download script and a checksum, and pin the dataset version in the run manifest.
- Labels never reach scoring or the agents.
- Report external results in their own section, with their limits. They show how Tickmark behaves on data the team did not design. They do not show real-world fraud detection, and Tickmark claims no ISA 240 compliance.
- These are public datasets, not client data, so the rule that keeps ledger data in Japan (ADR-015) does not apply to them.

## Order of work

1. Done on 30 Sep 2026: the Schreyer lab dataset is downloaded and its licences checked. The Gronewald data still needs locating (layer 2).
2. After the journal-line format is specified: build the converters and the business profile for the Gronewald company.
3. After the rules are frozen: run the locked evaluation, then layers 2 and 3.
4. Write the results into the evaluation report, including failed, blocked and unexecuted cases.

## Sources

- M. Schreyer, T. Sattarov, D. Borth, A. Dengel and B. Reimer, [Detection of Anomalies in Large-Scale Accounting Data using Deep Autoencoder Networks](https://arxiv.org/abs/1709.05254), 2017; lab code at [GitiHubi/deepAI](https://github.com/GitiHubi/deepAI).
- J. Gronewald, A. M. Rombach, S. Stephan and P. Fettke, [Anomaly Detection in General Ledger Data: Results from a Hybrid Approach](https://arxiv.org/abs/2609.18228), 2024.
- J. Gronewald and P. Fettke, [Generating Synthetic Journal Entries for Audit Analytics: Method and Dataset](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=7211804), SSRN, 2026.
- E. Lopez-Rojas, [Synthetic Financial Datasets For Fraud Detection (PaySim)](https://www.kaggle.com/datasets/ealaxi/paysim1), Kaggle, CC BY-SA 4.0; the source of the Schreyer lab dataset.
- [AuditCopilot: Leveraging LLMs for Fraud Detection in Double-Entry Bookkeeping](https://arxiv.org/abs/2512.02726), which evaluates on the Gronewald et al. data.

---

*Created 2026-09-30.*
