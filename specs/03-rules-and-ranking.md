# Spec 03 · Rules and ranking

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned, and not the owner of spec 02 (GR-32) · **Prefix:** RNK · **ADRs:** ADR-001, ADR-003, ADR-016, ADR-022 · **Ground rules:** GR-01, GR-04 to GR-09, GR-11, GR-25, GR-27

The rules engine tests every accepted entry against the five criteria, scores the entries that trip at least one, ranks them, and selects up to 300 for the working paper. It reads a ledger snapshot ([spec 01](01-ledger-format.md)) and the run settings defined here. Case grouping (spec 04) and the working paper (spec 07) use its output. No model is involved: code decides (GR-01).

## 1. Purpose

The brief asks Tickmark to identify the entries that carry fraud risk under ISA 240, rank them so that the top of the list is where a reviewer should start, and say which criterion caught each one (brief §3). The client's reviewers will check that the same ledger and settings give the same ranking (brief §5). This spec turns the five criteria into exact, testable rules, and fixes the score, the order and the record that proves both.

The default settings start from the team's decision of 30 Sep 2026 ([client log](../docs/client-log.md) Q-01), made before spec 02's scenarios were designed, with the additions agreed on 7 Oct 2026 (§6). The owner reviews them against the brief and ISA 240, never against spec 02's scenarios (GR-32).

## 2. Scope

**In scope:**
- The run settings file.
- The five criteria as sub-tests, and what happens when a column they need is missing.
- The score, the ranking and the selection.
- The scoring record and its hash.

**Out of scope:**
- Case grouping, the evidence pack and the approval-limit rules that use the run settings' `approval_limit` (spec 04).
- Agents (spec 05), and how the working paper shows the results (spec 07).
- Baselines and metrics, such as random and value-based selection (spec 10).
- Weights calibrated to the client's risk framework (assumption A6). Until it arrives, every criterion weighs the same.

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| RNK-01 | Tickmark shall read a run's criteria, materiality, approval limit and review budget from a run settings file (§4.1). If the file is invalid, Tickmark shall not start the run. | GR-08 |
| RNK-02 | Tickmark shall accept money in the run settings only as decimal text, and convert it exactly to minor units in the engagement's currency. | GR-05 |
| RNK-03 | Tickmark shall test every accepted entry against every enabled sub-test (§4.2), using only the ledger snapshot and the run settings. | GR-01, GR-07 |
| RNK-04 | Tickmark shall treat an entry as tripping a criterion when at least one of that criterion's sub-tests trips, and as flagged when it trips at least one criterion. | ADR-001 |
| RNK-05 | For each sub-test an entry trips, Tickmark shall record the values compared and the threshold, so that the working paper can show the calculation without recomputing it. | GR-11 |
| RNK-06 | Where a sub-test needs a column or setting that the snapshot or the run settings lack, Tickmark shall mark the sub-test not testable, never trip it, and report it with the reason. | Spec 01 |
| RNK-07 | Tickmark shall attach to every flag its criterion's ISA 240 and SLAuS 240 references and editions, as pinned in the rule bundle (§4.3). | ADR-022 |
| RNK-08 | Tickmark shall run each sub-test as one versioned SQL file in the rule bundle, with DuckDB, over the snapshot. Any change to a file shall need a new bundle version. | ADR-003, ADR-016 |
| RNK-09 | Tickmark shall compute each flagged entry's score in integers (§4.4): the sum of the weights of the criteria it trips, multiplied by the materiality multiplier when its amount is at or above performance materiality. | GR-05, client log Q-01 |
| RNK-10 | Tickmark shall rank flagged entries by score, highest first, then by amount, largest first, then by `entry_id` in byte order. | GR-06 |
| RNK-11 | Tickmark shall select the first N ranked entries, where N is the review budget, 300 by default. When fewer entries are flagged, it shall select all of them and record the actual cut-off. | Brief §1, scope §5 |
| RNK-12 | Tickmark shall never flag or select an entry that trips no criterion, whatever its amount. | ADR-001 |
| RNK-13 | Tickmark shall write a scoring record (§4.5) for every accepted entry, in a canonical form whose SHA-256 hash becomes part of the run's decision hash. | GR-04, ADR-003 |
| RNK-14 | Tickmark shall score a 400,000-entry snapshot and make its selection within 5 minutes on a cloud worker. | GR-27 |

## 4. Data shapes

### 4.1 Run settings: `run_settings.json`

Settings change between runs of the same snapshot, for example when the team refines the criteria. Each change starts a new run (GR-09).

| Field | Type | Default | Rule |
|---|---|---|---|
| `format_version` | Whole number | `1` | |
| `criteria.C1` to `criteria.C5`: `enabled` | `true` or `false` | `true` | A disabled criterion never trips. |
| `criteria.C1` to `criteria.C5`: `weight` | Decimal text | `"1.0"` | From 0 to 10, with at most 3 decimal places. |
| `criteria.C1.max_uses` | Whole number | `3` | At least 1. |
| `criteria.C1.classes` | List of spec 01 account classes | `["suspense_clearing"]` | |
| `criteria.C1.max_pair_journals` | Whole number | `3` | 0 turns C1-P off. |
| `criteria.C2.min_journals` | Whole number | `10` | At least 1. |
| `criteria.C2.manual_by_system_or_it` | `true` or `false` | `true` | Turns sub-test C2-M on or off. |
| `criteria.C2.unknown_user` | `true` or `false` | `true` | Turns sub-test C2-U on or off. |
| `criteria.C3.multiple_of` | Money | `"50000.00"` | Greater than zero. |
| `criteria.C3.at_least` | Money | `"1000000.00"` | |
| `criteria.C4.last_days_of_year` | Whole number | `5` | From 0 to 31, where 0 turns C4-Y off. |
| `criteria.C4.outside_working_hours` | `true` or `false` | `true` | Turns C4-H on or off. |
| `criteria.C4.non_working_days` | `true` or `false` | `true` | Turns C4-D on or off. |
| `criteria.C4.post_closing` | `true` or `false` | `true` | Turns C4-P on or off. |
| `criteria.C4.time_tests_manual_only` | `true` or `false` | `true` | C4-H and C4-D check only manual journals. |
| `criteria.C5.min_characters` | Whole number | `12` | From 0 to 100, where 0 turns C5-S off. |
| `criteria.C5.generic_words` | `true` or `false` | `true` | Turns C5-G on or off. |
| `materiality.overall`, `materiality.performance` | Money | From the profile | Performance materiality is at most overall materiality. |
| `materiality.multiplier` | Decimal text | `"1.5"` | From 1 to 10, with at most 3 decimal places. |
| `approval_limit` | Money | From the profile | Used by spec 04. |
| `review_budget` | Whole number | `300` | From 1 to 1,000. |

Money is decimal text in the engagement's currency, never a JSON number. Unknown fields make the file invalid.

```json
{
  "format_version": 1,
  "criteria": {
    "C1": { "enabled": true, "weight": "1.0", "max_uses": 3, "classes": ["suspense_clearing"], "max_pair_journals": 3 },
    "C2": { "enabled": true, "weight": "1.0", "min_journals": 10, "manual_by_system_or_it": true, "unknown_user": true },
    "C3": { "enabled": true, "weight": "1.0", "multiple_of": "50000.00", "at_least": "1000000.00" },
    "C4": { "enabled": true, "weight": "1.0", "last_days_of_year": 5, "outside_working_hours": true, "non_working_days": true, "post_closing": true, "time_tests_manual_only": true },
    "C5": { "enabled": true, "weight": "1.0", "min_characters": 12, "generic_words": true }
  },
  "materiality": { "overall": "25000000.00", "performance": "18750000.00", "multiplier": "1.5" },
  "approval_limit": "1000000.00",
  "review_budget": 300
}
```

### 4.2 Sub-tests

Each criterion is made of sub-tests. Counts cover the accepted rows of the snapshot, which spans one fiscal year (spec 01). Local dates and times use the engagement's timezone, and working days, working hours, holidays and close dates come from `engagement.json`.

| Sub-test | Trips when | Needs | Setting |
|---|---|---|---|
| **C1-R** Rarely used account | The entry's account appears in no more than `max_uses` journals. | | `max_uses` |
| **C1-S** Suspense or clearing account | The account's class is in `classes`. | A class other than `unclassified` | `classes` |
| **C1-P** Rare account pairing | The entry's journal debits an account of one class and credits an account of a different class, and that ordered pair of classes appears in no more than `max_pair_journals` journals. Every line of the journal whose class is in the rare pair trips. | Classes other than `unclassified` | `max_pair_journals` |
| **C2-R** Occasional poster | The journal's user posted fewer than `min_journals` journals. | `user_id` | `min_journals` |
| **C2-M** System or IT account used by hand | The user's type is `system` or their role is `it`, and the journal's source is `manual`. | `user_id`, `source` | `manual_by_system_or_it` |
| **C2-U** Unknown user | The user is missing from `users.csv` (spec 01's `UNKNOWN_USER` warning). | `user_id` | `unknown_user` |
| **C3** Round sum | The amount is a whole multiple of `multiple_of` and at least `at_least`. | | `multiple_of`, `at_least` |
| **C4-Y** Year end | `posting_date` falls in the last `last_days_of_year` days of the fiscal year, its end date included. | `posting_date` | `last_days_of_year` |
| **C4-H** Outside working hours | `entered_at`, in local time, is before the start of working hours, or at or after their end. With `time_tests_manual_only` on, only journals whose source is `manual` are checked. | `entered_at`, and `source` for the manual-only check | `outside_working_hours`, `time_tests_manual_only` |
| **C4-D** Non-working day | The local date of `entered_at` is not a working day, or is a listed holiday. With `time_tests_manual_only` on, only journals whose source is `manual` are checked. | `entered_at`, and `source` for the manual-only check | `non_working_days`, `time_tests_manual_only` |
| **C4-P** Post-closing | The local date of `entered_at` is after the close date of the month that `posting_date` falls in. | `entered_at`, `posting_date`, that month's close date | `post_closing` |
| **C5-E** Empty narration | The narration is empty or only spaces. | `narration` | Always on with C5 |
| **C5-S** Short narration | The narration, without leading and trailing spaces, has fewer than `min_characters` characters. | `narration` | `min_characters` |
| **C5-G** Generic narration | The narration has no word that is missing from the generic word list (§4.3). A narration with no words, such as `12/03`, also trips. | `narration` | `generic_words` |

**Words:** a word is a run of letters, lower-cased. Digits, punctuation and spaces separate words.

**Not testable:** a sub-test is not testable for the whole run when a column it needs is absent (spec 01, FMT-10). It is not testable for a single entry when that entry lacks what the sub-test needs, such as an account of class `unclassified` for C1-S, or a month with no close date for C4-P. When `source` is absent, C4-H and C4-D check every journal instead of manual journals only, and the scoring record says so. The scoring record lists every not-testable sub-test, with the reason and the number of entries affected.

### 4.3 Rule bundle

The rule bundle is versioned, such as `rules-1.0.0`, and pinned in the run manifest (GR-08). It holds:
- one SQL file per sub-test, and the scoring query;
- the generic word list;
- the criteria table below, with each criterion's ISA 240 and SLAuS 240 references and editions;
- a list of every file's SHA-256 hash, which CI checks against the version.

| Criterion | The brief's wording | ISA 240's characteristic of fraudulent journal entries | Notes |
|---|---|---|---|
| C1 | Entries to unrelated or unusual accounts | Entries made to unrelated, unusual or seldom-used accounts | Covers seldom-used accounts, suspense or clearing accounts, and rare pairings of account classes (C1-P). |
| C2 | Entries posted by people who do not normally post | Entries made by individuals who typically do not make journal entries | |
| C3 | Round-sum entries | Entries containing round numbers or consistent ending numbers | Consistent ending numbers are not tested, a known limitation (§6). |
| C4 | Entries at period end or outside business hours | Entries recorded at the end of the period, or as post-closing entries | Working hours and non-working days come from the brief, not from ISA 240's list. |
| C5 | Entries with weak or missing narration | Entries with little or no explanation or description | |
| — | — | Entries without account numbers | Rejected by spec 01 and listed in the validation report. |

The bundle cites the edition of ISA 240 in force for periods beginning before 15 December 2026: ¶32(a), the requirement to test journal entries and other adjustments, and ¶A41–A44, its application material on journal entries, where ¶A43 lists these characteristics. SLAuS 240 uses the same numbering, because CA Sri Lanka says its standards follow the structure of the ISAs. These numbers were proposed without checking the official text, so the audit adviser confirms them in both standards before the locked evaluation (ADR-022).

**Generic word list, version 1:** `acc`, `account`, `acct`, `adj`, `adjust`, `adjustment`, `agreed`, `amount`, `amt`, `and`, `approved`, `as`, `bal`, `balance`, `being`, `clearing`, `correction`, `discussed`, `entry`, `for`, `instructed`, `instruction`, `instructions`, `je`, `jnl`, `journal`, `misc`, `miscellaneous`, `na`, `of`, `other`, `payment`, `per`, `pmt`, `posting`, `reclass`, `request`, `requested`, `sundry`, `suspense`, `temp`, `test`, `tfr`, `the`, `to`, `transfer`, `trf`, `various`. Any one-letter word also counts as generic, which covers "a/c" and "n/a". The audit adviser checks the list before the locked evaluation.

### 4.4 Score

All arithmetic is in integers, so no result depends on the order of summing (GR-05):
- each weight becomes thousandths, so `"1.0"` is 1,000 and `"1.25"` is 1,250;
- the base score is the sum of the weights, in thousandths, of the criteria the entry trips;
- the multiplier becomes thousandths, so `"1.5"` is 1,500;
- the final score, in millionths, is the base score times the multiplier when the amount is at or above performance materiality, and times 1,000 otherwise.

For example, an entry that trips C1, C3 and C4 at the default weights has a base score of 3,000. At or above performance materiality, its final score is 4,500,000 (4.5); below it, 3,000,000 (3.0).

The ranking order is score, highest first, then amount, largest first, then `entry_id` in byte order. Amount comes before `entry_id` so that ties between equal scores favour larger entries rather than whichever ID happens to sort first.

### 4.5 Scoring record

The scoring record has three parts, each canonical text with LF line endings:
1. **Entries:** one row for every accepted entry, sorted by `entry_id`. Each row gives `entry_id`, `score`, `criteria` (such as `C1;C3`), `subtests` (such as `C1-R;C3`), and `values`, which holds every compared value and threshold, such as `C1-R:journals=2,max=3;C3:amount=500000000,multiple=5000000,min=100000000`. Unflagged entries have a score of 0 and empty fields.
2. **Selection:** `rank`, `entry_id` and `score` for every selected entry, in rank order, and the cut-off used.
3. **Testability:** each sub-test's status (`run`, `off`, `not_testable` or `partly_testable`), its reason, and the number of entries affected.

Amounts are in minor units, dates are `YYYY-MM-DD`, and times are local `HH:MM:SS`. The SHA-256 hash of the three parts, framed as in spec 01 §4.11, is the scoring hash. Spec 08 combines it with spec 04's case and evidence references into the run's decision hash.

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| RNK-T01 | A 400,000-entry development ledger and the default settings | It is scored | Every entry has a row, the flagged count is reported, 300 entries are selected, and it finishes within 5 minutes. *Ordinary* | RNK-03, RNK-04, RNK-11, RNK-14 |
| RNK-T02 | A fixture with one entry on each side of every threshold: 3 and 4 journals on an account, 9 and 10 journals by a user, amounts of 1,000,000.00, 1,050,000.00, 950,000.00 and 1,000,000.01, times of 07:59:59, 08:00:00, 17:59:59 and 18:00:00, posting dates 4 and 5 days before year end, and narrations of 11 and 12 characters | It is scored | Exactly the expected sub-tests trip, and each tripped sub-test records its values and threshold. *Awkward: period-boundary timestamps, amount precision* | RNK-03, RNK-04, RNK-05 |
| RNK-T03 | Narrations "Being payment", "Adj", "12/03", "" and "Payment to supplier S-0147 for invoice 4471" | They are scored | The first four trip C5. The last does not. | RNK-03, RNK-04 |
| RNK-T04 | Entries with equal scores and equal amounts, with IDs `A1`, `A10` and `A9` | They are ranked | The order is `A1`, `A10`, `A9`, which is byte order. *Awkward: tied scores* | RNK-10 |
| RNK-T05 | The same snapshot and settings, with rows shuffled, 1 and then 8 threads, and a worker stopped and restarted mid-scoring | It is scored each way | The scoring hash is identical every time. *Recovery* | RNK-13 |
| RNK-T06 | A snapshot without `user_id`, `entered_at`, `narration` and `posting_date`, like the Schreyer dataset | It is scored | C1 and C3 run. C2, C4 and C5 are reported not testable and never trip. | RNK-06 |
| RNK-T07 | An `engagement.json` without close dates, and then one missing a single month | It is scored | C4-P is not testable for the whole run in the first case, and only for that month's entries in the second, with the counts reported. | RNK-06 |
| RNK-T08 | A fixture with 40 flagged entries, and then one with none | It is scored | All 40 are selected with the cut-off recorded as 40, and then an empty selection is reported. *Awkward: small and empty populations* | RNK-11 |
| RNK-T09 | An entry above performance materiality that trips no criterion, and a flagged entry above it | They are scored | The first is not flagged or selected. The second's score is exactly 1.5 times its base score. | RNK-09, RNK-12 |
| RNK-T10 | An entry by a user missing from `users.csv` | It is scored | C2-U trips, and the record names the unknown user. | RNK-03, RNK-05 |
| RNK-T11 | Run settings with money as a JSON number, a weight with 4 decimal places, or an unknown field | A run is requested | The run does not start, and the error names the problem. | RNK-01, RNK-02 |
| RNK-T12 | A scored run | Its flags are read | Every flag carries its criterion's ISA 240 and SLAuS 240 references and editions from the bundle. | RNK-07 |
| RNK-T13 | A rule bundle where one SQL file changed but the version did not | CI runs | CI fails on the hash mismatch. | RNK-08 |
| RNK-T14 | The same snapshot scored twice, with `at_least` changed the second time | Both runs finish | They are separate runs, the first run's record is unchanged, and the selections differ as expected. | RNK-01, GR-09 |
| RNK-T15 | The same inputs, with the system clock moved a year ahead and a label folder beside the snapshot | It is scored | The scoring hash is unchanged, and the scoring process cannot open the label folder. | GR-07, GR-25 |
| RNK-T16 | A fixture where one ordered pair of account classes appears in 3 journals, another in 4, and a third involves an `unclassified` account | It is scored | The lines in the first pair trip C1-P and record the pair, its count and the limit. The second pair does not trip, and the third is reported as not testable. | RNK-03, RNK-05, RNK-06 |
| RNK-T17 | Two journals entered at 23:00 on a working day, one `manual` and one `automated`, and then the same journals without a `source` column | They are scored | With `source`, only the manual journal trips C4-H. Without it, both trip, and the scoring record says every journal was checked. | RNK-03, RNK-06 |
| RNK-T18 | Narrations "As per instructions", "As discussed" and "N/A" | They are scored | All three trip C5-G. | RNK-03 |

## 6. Open questions

None.

**Answered on 7 Oct 2026 (team decisions):**
1. Unrelated accounts: yes. Sub-test C1-P flags rare pairings of account classes, by default those found in no more than 3 journals a year.
2. Consistent ending numbers: not now. They are a known limitation, because the brief asks only for round sums, and legitimate prices and fees often repeat the same endings.
3. Time checks: C4-H and C4-D check only manual journals, because scheduled system jobs often post outside office hours. Without a `source` column, they check every journal and the record says so.
4. Post-closing: C4-P stays, on by default, because ISA 240 names post-closing entries.
5. Second sort key: amount stays. Counting sub-tests instead would favour criteria whose sub-tests overlap, such as an empty narration, which trips three C5 sub-tests.
6. Generic word list: version 1 in §4.3, with the added words and the one-letter rule. The audit adviser checks it before the locked evaluation.
7. Paragraph numbers: ISA 240 ¶32(a) and ¶A41–A44, with the characteristics in ¶A43, and the same numbers in SLAuS 240. The audit adviser confirms them before the locked evaluation.

---

*Created 2026-10-07.*
