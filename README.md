# Tickmark

**Evidence-Traceable Journal Entry Risk Review: Rules-First Ranking with Multi-Agent Justification for ISA 240 Testing**

> *Every flag ships with its tickmark.*

A tickmark is the symbol an auditor puts on a working paper to record which verification procedure was performed on an item — which is exactly what this platform produces for every entry it flags.

---

## One line

Tickmark reads a client's general ledger and tells the engagement team which journal entries deserve a human — and for every entry it flags, it names the criterion that caught it and shows the underlying record.

## For an industry audience (~30 seconds)

A typical client ledger at a mid-tier audit firm runs to 400,000 entries, and an engagement team can review about 300. Under ISA 240 they must test journal entries for fraud risk, and today they pick those 300 with crude filters and instinct. It takes two to three days, and when a regulator asks why those 300, the answer is weak.

Tickmark scores every entry with rules and statistics and picks the 300 that most deserve review. A router agent then sends each of those 300 to the specialist agents it needs, for the accounts and amounts, who posted it and when, and the narration. A writer agent turns their findings into a plain-English justification citing the ISA 240 criterion and the source record. The client's targets: under four hours and under forty dollars per engagement.

It doesn't replace the auditor. Tickmark ranks; the auditor concludes.

## For a technical audience (~45 seconds)

Tickmark is a rules-first batch pipeline with a routed team of AI agents at the narrow end of the funnel.

The interesting engineering problem is cost. The client allows forty dollars of compute and model calls per 400,000-entry engagement, and sending every row to a language model would exceed that by orders of magnitude. So queries, rules and statistics score every entry, and only the entries that make the working paper reach the agents. A router agent sends each one to the specialists it needs (account and amount, poster and timing, narration), and a code guard makes sure every flagged criterion gets its specialist. The specialists work in parallel from a shared case file, using a typed catalogue of read-only ledger queries, a writer drafts the justification and shows any disagreement side by side, and a code verifier rejects any draft that cites a figure or entry the evidence doesn't contain. We report exactly how many entries and model calls reached the agents, and why.

The client's reviewers will re-run the same ledger and expect the same ranking, so the ranking comes from deterministic criteria under a pinned run manifest, never from the agents. Routing decisions and agent outputs are recorded and replayed, so a re-run reproduces the working paper too, and if the model is down every flag still ships with its criterion, source record and a templated justification. Each run must finish in under four hours with twelve engagements running concurrently at peak, and ledgers must never leave the client's region.

---

**Status:** design stage. The figures above are the client's requirements, not measured results.

**Scope:** one audit procedure, done properly — ISA 240 journal entry testing, as a batch system that produces a working paper. Not a full audit platform, and not a verdict on whether fraud occurred. Constraints, boundaries and assessment mapping: [docs/scope.md](docs/scope.md).

**Diagrams:** [architecture](docs/diagrams/architecture.md) shows one batch run end to end, and [entry investigation](docs/diagrams/entry-investigation.md) shows how the routed agent team handles one flagged entry.

**Decisions:** the [architecture decision record](docs/adr.md) lists every significant choice, the alternatives we rejected and the assumptions we are working on until the client answers.

## Rules for explaining it

- Say **"reduces the review population"**. Never say "replaces the auditor".
- To assessors, lead with the **working batch run and its evidence**: runtime, cost per engagement, precision and recall. To industry, lead with the **pain**. Same system, two registers.
- Every flag names the **ISA 240 criterion** that caught it and links to its source record. Thresholds are our documented heuristics and weights follow the client's risk framework; neither is prescribed by the standard.
- **Code decides, agents explain.** Rules and statistics choose and rank the entries; the agents investigate and justify them but never change the order.

---

*Title and description last revised 2026-09-24 against the client brief (v1.0, 18 Sep 2026).*
