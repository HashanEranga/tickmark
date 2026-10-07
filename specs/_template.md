# Spec NN · Component name

**Status:** Draft, D Mon YYYY · **Owner:** name · **Prefix:** XXX · **ADRs:** ADR-0NN · **Ground rules:** GR-NN

One paragraph: what this component does, what it reads, what it produces and which component uses its output.

<!--
How to use this template (see specs/00-ground-rules.md):
- Copy this file to specs/NN-short-name.md and replace every placeholder.
- Keep the six numbered sections, in this order. Write "None." in a section that is empty.
- Each requirement is one testable sentence in EARS form, with one "shall":
    always:        Tickmark shall ...
    on an event:   When ..., Tickmark shall ...
    in a state:    While ..., Tickmark shall ...
    unwanted:      If ..., then Tickmark shall ...
    an option:     Where ..., Tickmark shall ...
- Do not repeat the ground rules. Cite them by ID where they apply.
- Use the terms from the glossary in the ground rules, with the same meaning.
- Every requirement needs at least one acceptance test, and every test names the requirements it checks.
- Tests use synthetic data only.
- Delete this comment before the pull request.
-->

## 1. Purpose

Why this component exists and the problem it solves, in two or three sentences. Link the scope section and ADRs it serves.

## 2. Scope

**In scope:**
- …

**Out of scope:**
- … (and which spec covers it instead)

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| XXX-01 | Tickmark shall … | ADR-0NN |
| XXX-02 | When …, Tickmark shall … | GR-NN |
| XXX-03 | If …, then Tickmark shall … | Scope §N |

## 4. Data shapes

The files, tables, records or messages this component reads and writes: each field's name, type, whether it is required, and its rules. Give one short example.

| Field | Type | Required | Rule |
|---|---|---|---|
| … | … | … | … |

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| XXX-T01 | … | … | … | XXX-01 |
| XXX-T02 | … | … | … | XXX-02, XXX-03 |

Mark tests that also serve as named evaluation cases (scope §5) as *ordinary*, *awkward*, *hostile* or *recovery* in the "Then" column.

## 6. Open questions

| # | Question | Who answers | Needed by |
|---|---|---|---|
| 1 | … | … | … |

---

*Created YYYY-MM-DD.*
