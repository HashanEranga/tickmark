# Spec 06 · Verifier and templates

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** VER · **ADRs:** ADR-010, ADR-011, ADR-013 · **Ground rules:** GR-11 to GR-13, GR-19

The verifier is code that checks every draft from the writer agent ([spec 05](05-agent-team.md)) before it can reach the working paper. A draft that fails gets one repair attempt. If that also fails, or no agent draft exists, code builds a template justification from the criteria and the evidence. Every flag therefore ships with its criterion, source record and calculations, whatever the agents do (GR-19).

## 1. Purpose

The client says a flag it cannot explain is worse than no flag, and every flag must resolve to a source record and a named criterion (brief §§5, 11). A language model can cite a figure that is not in the evidence, or state a conclusion an auditor must draw. The verifier catches both with fixed, repeatable checks rather than another model (ADR-011), and the template guarantees an explanation when no draft passes.

## 2. Scope

**In scope:**
- The checks on each draft, and the repair loop.
- The template justifications.
- The outcome recorded for every entry, and the rates reported per run.

**Out of scope:**
- Writing drafts (spec 05).
- Presenting justifications in the working paper (spec 07).
- Human review of justification quality (spec 10).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| VER-01 | Tickmark shall run every check in §4.1 on every draft, in code, with no model call. | ADR-011 |
| VER-02 | Tickmark shall reject a draft in which any cited evidence ID is missing from the case file. | GR-12 |
| VER-03 | Tickmark shall reject a draft in which any number, date or time in a justification does not match a value in the evidence that the justification cites. | GR-12 |
| VER-04 | Tickmark shall reject a draft that does not name, in each justification, every criterion that its entry trips. | GR-11 |
| VER-05 | Tickmark shall reject a draft in which any sentence matches a pattern on the conclusion list (§4.2). | GR-13 |
| VER-06 | Tickmark shall reject a draft that mentions an entry or journal outside its case and the evidence it cites, or that does not cover every entry of the case exactly once. | ADR-011 |
| VER-07 | When the case's findings disagree, Tickmark shall reject a draft that does not set `disagreement` and show both findings. | ADR-009 |
| VER-08 | If a draft fails, then Tickmark shall send the writer one repair request listing every failed check, when the allowance has room, and verify the repaired draft the same way. | ADR-010, ADR-011 |
| VER-09 | If no draft passes, or no draft exists, then Tickmark shall build a template justification for each of the case's entries (§4.3). | GR-19 |
| VER-10 | Tickmark shall build templates only from spec 03's values and spec 04's facts, so that every template passes the checks in §4.1. | ADR-011 |
| VER-11 | Tickmark shall record each entry's outcome (§4.4) and label each justification with its source: the agent's model and prompt version, or the template version. | Scope §5 |
| VER-12 | Tickmark shall report, for each run, the first-time pass rate, the repair success rate and the template rate by reason. | ADR-011 |
| VER-13 | Tickmark shall pin the version of the checks, the conclusion list and the templates in the run manifest. | ADR-003, GR-08 |
| VER-14 | Tickmark shall verify 300 justifications within 10 seconds on a cloud worker. | GR-27 |

## 4. Data shapes

### 4.1 Checks

| Check | A justification fails when |
|---|---|
| V1 · Evidence exists | It cites an evidence ID that is not in the case file. |
| V2 · Numbers match | A number in its text does not equal a value in the evidence it cites. |
| V3 · Dates and times match | A date or time in its text does not equal a value in the evidence it cites. |
| V4 · Criteria named | It does not name, by code such as `C3`, every criterion its entry trips, or `criteria_addressed` misses one. |
| V5 · No conclusion | A sentence matches the conclusion list (§4.2). |
| V6 · Right entries | It mentions an entry or journal ID that is neither in the case nor in its cited evidence, or the draft does not cover each case entry exactly once. |
| V7 · Disagreement shown | The findings disagree, but `disagreement` is false, the note is empty, or the text does not contain "specialists disagree". |

**Reading numbers, dates and times:**
- **Amounts:** written as in the evidence, such as `LKR 1,250,000.00`. Commas are ignored when comparing, and an amount must match to the minor unit.
- **Counts:** whole numbers, compared exactly.
- **Dates:** `2026-03-31`, `31 Mar 2026` or `31 March 2026`.
- **Times:** 24-hour, `21:14` or `21:14:05`.
- **Ignored:** criterion codes, evidence IDs, and entry, journal and account IDs, which V1 and V6 check instead.
- **Not checked:** numbers written as words, which the writer's prompt forbids. The human review in spec 10 samples for them.

### 4.2 Conclusion list, version 1

Case-insensitive patterns that state or imply that fraud occurred:
- `is fraud`, `is fraudulent`, `was fraud`, `was fraudulent`, `are fraudulent`, `were fraudulent`;
- `fraud occurred`, `fraud has occurred`, `fraud was committed`, `committed fraud`, `perpetrated`;
- `confirms fraud`, `confirmed fraud`, `proves`, `proof of fraud`;
- `embezzled`, `embezzlement`, `stole`, `stolen`, `guilty`.

Phrases such as "may indicate a risk of", "warrants review" and "is consistent with" are allowed. A false alarm on this list only sends the entry to the template, which is the safe direction.

### 4.3 Templates, version 1

Each entry's template has three parts:
1. **Opening:** "Entry {entry_id} (journal {journal_id}, posted {posting_date} to account {account_id} {account_name}, LKR {amount}) was selected because it trips {criteria, by code and name}."
2. **One sentence per sub-test the entry trips:**

   | Sub-test | Sentence |
   |---|---|
   | C1-R | Account {account_id} was used in {journals} journals this year, within the limit of {max}. |
   | C1-S | Account {account_id} is a {class} account. |
   | C1-P | The journal pairs {debit_class} with {credit_class}, a pairing found in {journals} journals this year, within the limit of {max}. |
   | C2-R | User {user_id} posted {journals} journals this year, fewer than {min}. |
   | C2-M | User {user_id} is a {user_type} account with the {role} role, and the journal was keyed by hand. |
   | C2-U | User {user_id} is not in the user list. |
   | C3 | The amount, LKR {amount}, is a multiple of LKR {multiple} and at least LKR {min}. |
   | C4-Y | It was posted on {posting_date}, within the last {days} days of the year. |
   | C4-H | It was keyed at {time}, outside working hours of {start} to {end}. |
   | C4-D | It was keyed on {date}, which is {a weekend day or a listed holiday}. |
   | C4-P | It was keyed on {date}, {days} days after its period closed on {close_date}. |
   | C5-E | Its narration is empty. |
   | C5-S | Its narration has {length} characters, fewer than {min}. |
   | C5-G | Its narration contains only generic words. |

3. **Closing:** "{It is reviewed with {n} related entries in {case_id}, linked by {rules}.} This is a reason to review the entry, not a finding of fraud."

Templates leave out any part whose value is absent, such as a posting date in a ledger without that column. They cite the evidence IDs of the facts they use.

### 4.4 Entry outcomes

| Outcome | Meaning |
|---|---|
| `agent` | The first draft passed. |
| `agent_repaired` | The repaired draft passed. |
| `template_verifier` | The draft failed and the repair also failed, or there was no room for a repair. |
| `template_schema` | The writer's output failed its schema twice. |
| `template_model_unavailable` | Model calls failed after their retries (spec 05). |
| `template_allowance` | The case's allowance ran out first. |
| `template_budget` | The run's worst case exceeded the budget even after the cuts (spec 05). |
| `template_deadline` | The run reached its deadline fallback (spec 08). |

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| VER-T01 | 300 drafts from a development run | They are verified | Each gets an outcome, the rates are reported, and it finishes within 10 seconds. *Ordinary: every justification passes or falls back* | VER-01, VER-11, VER-12, VER-14 |
| VER-T02 | A draft citing `CASE-0003-E099`, which does not exist | It is verified | It fails V1. | VER-02 |
| VER-T03 | A draft stating "LKR 1,250,000.01" where the evidence holds 1,250,000.00, and another stating a count of 4 where the evidence holds 3 | They are verified | Both fail V2, naming the number. | VER-03 |
| VER-T04 | A draft giving the date 30 Mar 2026 where the evidence holds 2026-03-31, and a time of 21:15 where it holds 21:14 | They are verified | Both fail V3. | VER-03 |
| VER-T05 | A draft for an entry tripping C1 and C3 that names only C1 | It is verified | It fails V4. | VER-04 |
| VER-T06 | Drafts containing "this entry is fraudulent", "fraud was committed" and "proves" | They are verified | Each fails V5, and a draft saying "may indicate a risk of misstatement" passes it. | VER-05 |
| VER-T07 | A draft that mentions a journal from another case, and one that skips an entry | They are verified | Both fail V6. | VER-06 |
| VER-T08 | Disagreeing findings, and a draft that shows only one of them | It is verified | It fails V7. *Awkward: specialists that disagree* | VER-07 |
| VER-T09 | A draft failing V2, then a repaired draft that passes | The repair loop runs | One repair request lists the V2 failure, and the outcome is `agent_repaired`. | VER-08 |
| VER-T10 | A draft that fails twice | The repair loop runs | Every entry in the case gets a template, with outcome `template_verifier`. | VER-08, VER-09 |
| VER-T11 | The templates for every sub-test, built from fixture values | They are verified | Every template passes every check, and each cites its evidence IDs. | VER-10 |
| VER-T12 | A case with no draft because the model was unavailable | The case closes | Each entry gets a template with outcome `template_model_unavailable`, labelled with the template version. *Recovery: model-provider outage* | VER-09, VER-11 |
| VER-T13 | A run manifest | It is read | It pins the checks', conclusion list's and templates' versions. | VER-13 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. A justification must name each tripped criterion by its code, such as `C3`, so that V4 is an exact check.
2. Numbers written as words are not checked; the writer's prompt forbids them, and the human review samples for them.
3. The conclusion list errs on the side of caution, because a false alarm only sends the entry to the template.
4. Drafts that disagree must contain the words "specialists disagree", which the working paper also shows.

---

*Created 2026-10-07.*
