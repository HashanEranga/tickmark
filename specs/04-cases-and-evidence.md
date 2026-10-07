# Spec 04 · Cases and evidence

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** CAS · **ADRs:** ADR-003, ADR-005, ADR-008, ADR-009, ADR-012, ADR-013 · **Ground rules:** GR-01, GR-02, GR-04, GR-11, GR-16, GR-17, GR-24

After the rules engine has selected up to 300 entries ([spec 03](03-rules-and-ranking.md)), code groups related selected entries into cases and builds each case's evidence pack: the facts the agents read, each with an evidence ID. This spec also defines the typed query catalogue, the only way an agent can ask the ledger for more. The agent team (spec 05) reads the case files, and the verifier (spec 06) checks justifications against the evidence IDs.

## 1. Purpose

A scheme often spans several entries, such as an entry and its reversal, or one payment split into parts. Investigating each entry alone repeats work and misses the link (ADR-012). Agents also need context, such as how an account is normally used, but letting them write their own queries would make results depend on what they happened to ask and open a path for injected narration (ADR-008). This spec makes both steps code: deterministic grouping, a fixed evidence pack, and a small catalogue of read-only queries.

## 2. Scope

**In scope:**
- Grouping the selected entries into cases.
- The evidence pack for each case, and its evidence IDs.
- The typed query catalogue that specialists may call.
- The case file's structure, and the case hash.

**Out of scope:**
- Which specialists a case gets, how many queries they may make, and the allowance (spec 05).
- Checking justifications against the evidence (spec 06).
- How cases appear in the working paper (spec 07).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| CAS-01 | Tickmark shall group only the selected entries into cases, using the grouping rules in §4.1 in their listed order, and shall never change the ranking or the selection. | ADR-012 |
| CAS-02 | Tickmark shall record for each case the rules that linked its entries. | GR-11 |
| CAS-03 | If a group would hold more than 12 entries, then Tickmark shall split it into cases of at most 12 entries, in rank order, and record that the cases belong together. | ADR-014 |
| CAS-04 | Tickmark shall number cases in the order of each case's best-ranked entry, as `CASE-0001` upwards. | GR-04 |
| CAS-05 | When a case opens, Tickmark shall build its evidence pack (§4.2) with fixed read-only queries chosen by the criteria its entries trip, before any agent reads it. | ADR-008 |
| CAS-06 | Tickmark shall give every fact in a case file an evidence ID, such as `CASE-0007-E012`, assigned in a fixed order. | ADR-011 |
| CAS-07 | Tickmark shall keep each evidence pack within 60 facts and 8,000 characters, dropping the lowest-priority facts first and recording what was dropped. | ADR-014 |
| CAS-08 | Tickmark shall mark every fact that holds ledger text, such as narration, as untrusted text, so that the agents receive it only as quoted data. | GR-16, ADR-013 |
| CAS-09 | Tickmark shall offer specialists only the queries in the catalogue (§4.3), each with validated parameters and a row limit. | ADR-008, GR-17 |
| CAS-10 | Tickmark shall run every catalogue query read-only, against the run's own ledger snapshot only, whatever parameters an agent supplies. | GR-17, GR-24 |
| CAS-11 | If a catalogue call has invalid parameters, then Tickmark shall return a typed error to the agent and run nothing. | ADR-008 |
| CAS-12 | Tickmark shall add every catalogue result to the case file as new facts with their own evidence IDs. | ADR-008 |
| CAS-13 | Tickmark shall write the cases and evidence packs in a canonical form whose SHA-256 hash, the case hash, becomes part of the run's decision hash (spec 08). | GR-04, ADR-003 |
| CAS-14 | Tickmark shall build the cases and evidence packs for 300 selected entries within 2 minutes on a cloud worker. | GR-27 |

## 4. Data shapes

### 4.1 Grouping rules

Rules apply only to selected entries. Entries linked by any rule, directly or through other entries, form one group.

| Rule | Links selected entries that… | Setting |
|---|---|---|
| G1 · Same journal | share a `journal_id` | |
| G2 · Reversal | are in a journal and in the journal it reverses, through `reverses_journal_id` | |
| G3 · Split under the approval limit | share an account, side and user, have posting dates no more than 3 days apart, are each below the approval limit, and together reach it | `approval_limit` (spec 03), a 3-day window |
| G4 · Same unusual poster | share a user and a posting month, where every linked entry trips C2 | |

G4 is limited to entries that trip C2, because grouping all of one busy user's entries would build large cases from unrelated work. Related entries that were not selected never join a case; they can appear in its evidence pack as facts.

### 4.2 Evidence pack

Facts are added in the order below, so a dropped fact is always one of the last. Within each group, facts follow `entry_id` order.

| Order | Included for | Facts |
|---|---|---|
| 1 | Every case | Each member entry's fields; every line of its journal; its account's name and class; its user's type and role; the sub-tests it tripped, with spec 03's values and thresholds; how it is linked to the other members |
| 2 | C1 | The account's journals per month this year; its first and last use; the 5 account classes it is most often paired with; the count for any rare class pair (C1-P) |
| 3 | C2 | The user's journals per month; the 5 accounts the user posts to most; the user's postings by hour of the day; the journal's source |
| 4 | C3 | The account's median and 90th-percentile amounts; how many round sums the account received this year; whether the amount reaches performance materiality |
| 5 | C4 | The local time and day type when the journal was keyed; its month's close date; how many days after the close it was keyed; how many of the user's journals were keyed outside working hours |
| 6 | C5 | The narration, as untrusted text; its length; the generic words found; up to 5 narrations from other journals on the same account, as untrusted text |
| 7 | Every case | Journals linked through `reverses_journal_id` that were not selected; other journals by the same user on the same day |

Each fact has this shape:

| Field | Meaning |
|---|---|
| `evidence_id` | Such as `CASE-0007-E012` |
| `kind` | The query that produced it, such as `account_activity` |
| `params` | The query's parameters |
| `value` | The result, as typed JSON with amounts in minor units |
| `entry_ids` | The ledger entries the fact came from |
| `untrusted_text` | `true` when `value` holds ledger text |

### 4.3 Query catalogue

Specialists call these through LangChain tools (ADR-017). Code runs each query on the run's snapshot, so a parameter can never reach another engagement. Every result is capped at 20 rows.

| Query | Parameters | Returns |
|---|---|---|
| `account_activity` | `account_id`, `from_month`, `to_month` | Journals and total amount per month for the account |
| `user_activity` | `user_id`, `from_month`, `to_month` | Journals per month, the accounts used, and postings by hour |
| `user_account_history` | `user_id`, `account_id` | The user's postings to the account this year, newest first |
| `related_journals` | `journal_id` | Journals linked by reversal, or sharing a `document_ref` |
| `class_pair_history` | `debit_class`, `credit_class` | How many journals pair the two classes, with up to 20 examples |
| `account_narrations` | `account_id` | Up to 20 narrations on the account, as untrusted text |
| `amount_profile` | `account_id` | Percentiles of amounts, and the count of round sums |
| `period_close` | `month` | The month's close date, and the journals keyed after it |

**Validation:** IDs must exist in the snapshot; months must be in the fiscal year as `YYYY-MM`, with `from_month` no later than `to_month`; classes must be spec 01 classes. Any other parameter is an error.

### 4.4 Case file and case hash

The case file lives in the run store (PostgreSQL, ADR-016). This spec defines its first parts; specs 05 and 06 add the routing decision, findings, draft and verification.

| Part | Content |
|---|---|
| Case | `case_id`, member `entry_ids` in rank order, the linking rules, and any sibling cases from a split |
| Evidence pack | The facts in §4.2, and any dropped-fact note |
| Catalogue results | The facts added by catalogue calls, each linked to the call that produced it |

The case hash is the SHA-256 of the cases and evidence packs, written as canonical JSON (RFC 8785), one case per line in case order. Catalogue results are left out, because they depend on what the agents asked; the replay store keeps them repeatable instead (ADR-010).

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| CAS-T01 | 300 selected entries from a development ledger | Cases are built | Every selected entry is in exactly one case, the ranking is unchanged, and it finishes within 2 minutes. *Ordinary* | CAS-01, CAS-14 |
| CAS-T02 | Two selected lines of one journal, and a selected journal plus the journal it reverses | Cases are built | Each pair forms one case, recorded as G1 and G2. *Awkward: related entries grouped into one case* | CAS-01, CAS-02 |
| CAS-T03 | Three selected payments to one account by one user over 2 days, each below the approval limit and together above it, and a fourth 5 days later | Cases are built | The three form one case under G3, and the fourth stays apart. | CAS-01, CAS-02 |
| CAS-T04 | Fifteen selected entries by one occasional poster in one month, all tripping C2 | Cases are built | They form two cases of 12 and 3 in rank order, linked as siblings. | CAS-03, CAS-04 |
| CAS-T05 | A busy user's selected entries that do not trip C2 | Cases are built | G4 does not link them. | CAS-01 |
| CAS-T06 | The same selection built twice, with rows shuffled and on 1 and 4 threads | Cases are built | The case IDs, evidence IDs and case hash are identical. | CAS-04, CAS-06, CAS-13 |
| CAS-T07 | A case whose entries trip C1 and C5 | Its evidence pack is built | It holds the every-case facts, then C1 and C5 facts in the order of §4.2, each with an evidence ID, and narrations marked as untrusted text. | CAS-05, CAS-06, CAS-08 |
| CAS-T08 | A case whose facts would exceed 60 facts or 8,000 characters | Its evidence pack is built | The last-ordered facts are dropped and the pack records what was dropped. | CAS-07 |
| CAS-T09 | Catalogue calls with an unknown account, a month outside the year, a SQL fragment in a parameter, and another engagement's journal ID | They are called | Each returns a typed error, and no query runs. *Hostile: attempts to query another engagement's data* | CAS-09, CAS-10, CAS-11 |
| CAS-T10 | A valid `account_narrations` call | It is called | At most 20 rows return, as facts with new evidence IDs, marked as untrusted text. | CAS-08, CAS-12 |
| CAS-T11 | The catalogue's database role | A write is attempted through it | The database refuses the write. | CAS-10 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. G4 links only entries that trip C2, so a busy user's normal work does not form one large case.
2. A case holds at most 12 entries, and larger groups split in rank order.
3. An evidence pack holds at most 60 facts and 8,000 characters. That leaves room in spec 05's prompt limit of 12,000 characters for the instructions, the schema and the catalogue.
4. Catalogue results are capped at 20 rows, and are left out of the case hash because they depend on agent choices.

---

*Created 2026-10-07.*
