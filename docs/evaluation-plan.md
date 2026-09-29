# Evaluation plan

How Tickmark's detection and agent quality will be measured. The main evaluation uses the team's own full-scale synthetic ledgers, as the brief requires (brief §7); two public datasets are run as an external check. The metrics come from [scope.md](scope.md) §5, the decision from [ADR-004](adr.md#adr-004--full-scale-synthetic-ledgers-checked-against-public-datasets), and the pass thresholds from the [client log](client-log.md) (Q-02). The results go into the evaluation report, one of the handbook's six deliverables.

## Three layers

| Layer | Data | What it shows | Pass mark |
|---|---|---|---|
| 1 · Main | Tickmark's synthetic ledgers: two business profiles, 400,000 journal lines each | Detection and agent quality at full scale, against known answers | Yes, the team's thresholds below |
| 2 · External | Gronewald et al. (2024) synthetic ledger | How the frozen pipeline behaves on a ledger the team did not design; all five criteria can run | No, reported only |
| 3 · External | Schreyer et al. (2017) SAP ledgers | False alarms among real business entries; criterion C1, and C3 if amounts are present | No, reported only |

## Layer 1 · Tickmark's synthetic ledgers

**Business profiles.** Two fictional companies of different business types: retail first (scope §2), and a second chosen by the team. Each profile sets its chart of accounts, users and roles, working hours, holidays (including Sri Lankan public and Poya holidays), close calendar, approval limits and materiality, all in the same ledger format. If a profile draws on a real company's ledger, the team needs that company's written permission and uses only aggregate statistics, never its rows, names or narrations.

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

- **What it is:** a synthetic general ledger of a fictional medium-sized company, from Peter Fettke's research group, with labelled anomalous entries. Papers that use it report posting and transaction times, users, accounts, amounts, currency, tax rates, debit and credit flags, and memo text, so all five criteria can run. Confirm the fields once the data is in hand.
- **Get it:** find the download through the authors' papers, or ask the authors. If it cannot be obtained within one week, drop this layer and say so in the report.
- **Convert:** map it into Tickmark's journal-line format and its accounts to canonical account classes, and write a business profile for the fictional company.
- **Map labels:** link each anomaly type to the criteria that should catch it. Types that no criterion covers, such as a wrong tax rate, are reported as misses by design.
- **Run:** the frozen rule bundle from the locked evaluation, with no tuning on this data.
- **Measure:**
  - All flagged entries as true and false positives, comparable with the published rule-based baseline. On a 5,000-entry sample with 50 anomalies, rule-based journal entry tests caught all 50 but also flagged 942 innocent entries (AuditCopilot, Table 1).
  - The ranked list at k = 300, and at k equal to the number of labelled anomalies.
  - The agent team on the top of the list: verifier pass rate, template fallback rate and cost.

## Layer 3 · Schreyer et al. (2017)

- **What it is:** two anonymised SAP ledgers of real journal entries, with 307,457 lines (dataset A) and 172,990 lines (dataset B). The authors injected 95 and 100 synthetic anomalies and labelled every line as regular, a global anomaly (a rare value) or a local anomaly (a rare combination of values). The paper describes only categorical fields, 6 and 10 of them, so there are no users, times or narration.
- **Get it:** through the authors' published lab code ([GitiHubi/deepAI](https://github.com/GitiHubi/deepAI)); check which data it ships and under what licence.
- **Run:** criterion C1, plus C3 if the version we get includes amounts. C2, C4 and C5 are reported as not testable. The agents do not run, because the entries have no users or narration to investigate.
- **Measure:** false alarms among the real, regular entries, and recall of global and local anomalies in the top 300. Dataset A is close to Tickmark's 400,000 lines, so the 300-entry review budget applies unchanged.

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

1. Now: locate both datasets and check their licences.
2. After the journal-line format is specified: build the converters and the business profile for the Gronewald company.
3. After the rules are frozen: run the locked evaluation, then layers 2 and 3.
4. Write the results into the evaluation report, including failed, blocked and unexecuted cases.

## Sources

- M. Schreyer, T. Sattarov, D. Borth, A. Dengel and B. Reimer, [Detection of Anomalies in Large-Scale Accounting Data using Deep Autoencoder Networks](https://arxiv.org/abs/1709.05254), 2017; lab code at [GitiHubi/deepAI](https://github.com/GitiHubi/deepAI).
- J. Gronewald, A. M. Rombach, S. Stephan and P. Fettke, [Anomaly Detection in General Ledger Data: Results from a Hybrid Approach](https://arxiv.org/abs/2609.18228), 2024.
- [Generating Synthetic Journal Entries for Audit Analytics](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=7211804), SSRN, a related paper describing a labelled synthetic ledger.
- [AuditCopilot: Leveraging LLMs for Fraud Detection in Double-Entry Bookkeeping](https://arxiv.org/abs/2512.02726), which evaluates on the Gronewald et al. data.

---

*Created 2026-09-30.*
