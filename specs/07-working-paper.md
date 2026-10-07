# Spec 07 · Working paper

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** WPR · **ADRs:** ADR-002, ADR-009, ADR-010, ADR-013, ADR-022 · **Ground rules:** GR-08, GR-11, GR-13, GR-16, GR-33

Each completed run ends in a working paper: a PDF that an engagement reviewer can sign, with CSV appendices. It shows why each of the selected entries deserves review, with its criteria, source record, calculations, findings and justification, and how the population was established. It follows the 4-page sample the team shared on 29 Sep 2026 (assumption A5).

## 1. Purpose

The brief's definition of done asks for a working paper that an engagement reviewer would sign (brief §11). A reviewer signs only what they can follow and check, so every number in the paper traces back to the ledger, every flag names its criterion, and nothing claims that fraud occurred.

## 2. Scope

**In scope:**
- The PDF's sections and contents, and the CSV appendices.
- Formatting, escaping and how the paper is reproduced.

**Out of scope:**
- Signing. Tickmark leaves the sign-off block blank, and e-signatures are out of scope.
- Recording the auditor's conclusions, which happens on the signed paper, outside Tickmark (ADR-002).
- The checks behind the content (specs 01 to 06).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| WPR-01 | When a run succeeds, Tickmark shall produce one PDF and the CSV appendices in §4.2. A failed or cancelled run shall get no working paper. | ADR-002 |
| WPR-02 | Tickmark shall lay out the PDF in the sections of §4.1, in that order. | Assumption A5 |
| WPR-03 | For every selected entry, Tickmark shall show its criteria by code and name, its source record, the calculations behind each tripped sub-test, the specialists' findings and the justification. | Scope §2, GR-11 |
| WPR-04 | When specialists disagree, Tickmark shall show both findings side by side under the heading "Specialists disagree". | ADR-009 |
| WPR-05 | Tickmark shall label every justification with its source: the agent's model and prompt version, or the template version. | Scope §5 |
| WPR-06 | Tickmark shall include the submission's validation report: the counts, the control totals for reconciling to the trial balance, and every rejected row and warning in an appendix. | Spec 01 |
| WPR-07 | Tickmark shall state the standards applied, ISA 240 and SLAuS 240 with their editions and references, and that a flag is a reason to review, not a finding of fraud. | ADR-022, GR-13 |
| WPR-08 | Tickmark shall list every sub-test that was off or not testable, with its reason, and the known limitations. | GR-33 |
| WPR-09 | Tickmark shall include the run's provenance: the manifest hash, snapshot hash, settings hash and decision hash, and every pinned version and model ID. | GR-08 |
| WPR-10 | Tickmark shall end the PDF with a blank sign-off block for the preparer, the reviewer and the manager. | Assumption A5 |
| WPR-11 | Tickmark shall render narration and other ledger text as plain quoted text, escaping anything a renderer could read as markup. | GR-16, ADR-013 |
| WPR-12 | Tickmark shall protect the CSV appendices against spreadsheet formulas by prefixing any value that starts with `=`, `+`, `-`, `@`, a tab or a carriage return with a single quote. | ADR-013 |
| WPR-13 | Tickmark shall build the paper from a canonical working-paper document, so that the document and the CSV appendices are byte-identical when a run is reproduced, and the PDF differs only in its "generated at" time. | ADR-010, GR-04 |
| WPR-14 | Tickmark shall write amounts as `LKR 1,250,000.00`, dates as `31 Mar 2026`, and times in 24-hour form, stating the engagement's timezone. | Assumption A5 |
| WPR-15 | Tickmark shall store the working paper with its run, unchangeable, and keep it retrievable after a restart. A re-run creates its own paper and never overwrites an earlier one. | ADR-002, scope §2 |
| WPR-16 | Tickmark shall produce the working paper for 300 selected entries within 5 minutes on a cloud worker. | GR-27 |

## 4. Data shapes

### 4.1 PDF sections

| # | Section | Contents |
|---|---|---|
| 1 | Cover | Title, entity, fiscal year, engagement ID, run ID and its number among the engagement's runs, and "Synthetic data" where the ledger came from the generator |
| 2 | Purpose and basis | What Tickmark did, the standards applied (WPR-07), the five criteria with the brief's wording, and the statement that the auditor concludes |
| 3 | Population | The validation summary: rows received, accepted and rejected, counts by reason, warnings, control totals, and absent columns |
| 4 | Settings and testability | Thresholds, weights, materiality and review budget; each sub-test's status, with reasons and counts |
| 5 | Selection | Flagged and selected counts and the cut-off; counts by criterion; cases; entries that reached the agents and model calls made and replayed; justification outcomes by reason; measured and worst-case model cost |
| 6 | Selected entries | One row per selected entry: rank, entry, journal, posting date, account, amount, criteria, score, case and justification source |
| 7 | Cases | One block per case: its entries' source records, the linking rules, each entry's calculations, the cited evidence (narration as quoted text), the findings, any "Specialists disagree" block, and each entry's justification with its source label |
| 8 | Limitations | Sub-tests that were off or not testable, known limitations such as consistent ending numbers not being tested, and rejected rows not being ranked |
| 9 | Provenance | Hashes, pinned versions, model IDs, and the engagement's earlier runs with their decision hashes |
| 10 | Sign-off | Name, signature and date lines for the preparer, the reviewer and the manager, and a space for review notes |

Every page carries the entity, the run ID, the page number and "Prepared by Tickmark for review; not a finding of fraud" in its footer.

### 4.2 CSV appendices

| File | One row per | Columns |
|---|---|---|
| `selected_entries.csv` | Selected entry | `rank`, every `entries.csv` field, `criteria`, `subtests`, `values`, `score`, `case_id`, `justification_source` |
| `justifications.csv` | Selected entry | `entry_id`, `case_id`, `outcome`, `source`, `justification`, `evidence_ids` |
| `findings.csv` | Finding | `case_id`, `specialist`, `assessment`, `summary`, `points`, `follow_up` |
| `rejected_rows.csv` | Rejected row | As in spec 01 §4.10 |
| `warnings.csv` | Warning | As in spec 01 §4.10 |
| `testability.csv` | Sub-test | `subtest`, `status`, `reason`, `entries_affected` |
| `manifest.json` | Run | The run manifest (spec 08) |

CSV files are UTF-8 with a header row, comma-separated, quoted as RFC 4180 requires, with LF line endings. Narration is the original text apart from the formula prefix (WPR-12), and `entries.csv` in the snapshot keeps the exact original.

### 4.3 Working-paper document

The paper is rendered from one JSON document, written in RFC 8785 canonical form, that holds every value the PDF shows, in section order. The renderer embeds fixed fonts, sets fixed PDF metadata, and adds the "generated at" time on the cover only. The SHA-256 of the document is printed in the provenance section, so a reader can check that a PDF matches its run.

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| WPR-T01 | A succeeded development run with 300 selected entries | Its working paper is produced | The PDF has the sections in §4.1, in order, the appendices in §4.2 exist, and it finishes within 5 minutes. *Ordinary: working paper produced* | WPR-01, WPR-02, WPR-16 |
| WPR-T02 | A failed run and a cancelled run | They end | Neither has a working paper. | WPR-01 |
| WPR-T03 | The paper from WPR-T01 | Each selected entry is traced | Every entry shows its criteria, source record, calculations, findings and justification, and every number in it matches the ledger or the scoring record. | WPR-03 |
| WPR-T04 | A case whose specialists disagree | Its block is rendered | Both findings appear side by side under "Specialists disagree". | WPR-04 |
| WPR-T05 | A run with agent and template justifications | The paper is read | Each justification shows its source: model and prompt version, or template version. | WPR-05 |
| WPR-T06 | A run started from a submission with rejected rows | The paper is produced | The population section shows the counts and control totals, and `rejected_rows.csv` lists every rejected row. | WPR-06 |
| WPR-T07 | Any working paper | Its basis section and footers are read | It names ISA 240 and SLAuS 240 with editions, and states that a flag is not a finding of fraud. | WPR-07 |
| WPR-T08 | A run on a ledger without `narration` | The paper is produced | The limitations section lists C5 as not testable, with its reason. | WPR-08 |
| WPR-T09 | Any working paper | Its provenance section is read | It shows the manifest, snapshot, settings, decision and document hashes, and they match the run record. | WPR-09 |
| WPR-T10 | Any working paper | Its last section is read | A blank sign-off block for preparer, reviewer and manager is present. | WPR-10 |
| WPR-T11 | A narration of `<b>bold</b> =HYPERLINK("x")` | The paper and appendices are produced | The PDF shows the text literally, and the CSV value starts with a single quote. *Hostile: adversarial narration* | WPR-11, WPR-12 |
| WPR-T12 | A run reproduced from its stored inputs and outputs | The paper is produced again | The document and CSV files are byte-identical, and the PDF differs only in its "generated at" time. | WPR-13 |
| WPR-T13 | Amounts, dates and times in a fixture run | The paper is produced | They read `LKR 1,250,000.00`, `31 Mar 2026` and `21:14 (Asia/Colombo)`. | WPR-14 |
| WPR-T14 | A run, then a re-run with changed settings, then a worker restart | Both papers are fetched | Both exist unchanged, each with its own run ID. | WPR-15 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. A failed or cancelled run produces no working paper, so that a partial paper is never mistaken for a complete one.
2. CSV appendices prefix formula-like values with a single quote. The snapshot keeps the original text.
3. The PDF is rendered from a canonical JSON document, whose hash is printed in the paper.
4. Every footer says the paper is prepared for review and is not a finding of fraud.

---

*Created 2026-10-07.*
