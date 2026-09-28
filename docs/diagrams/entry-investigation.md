# Tickmark: investigating one flagged entry

Step ⑤ of the [architecture](architecture.md), using the same boxes, for one entry flagged on three criteria so that all three specialists are needed. The rules come from [scope.md §2](../scope.md), and the decisions behind them are ADR-007 to ADR-014 in [adr.md](../adr.md).

```mermaid
%%{init: {"sequence": {"width": 90, "actorMargin": 25}}}%%
sequenceDiagram
    participant R as Router<br/>agent
    participant G as Routing<br/>guard (code)
    participant A as Account and<br/>amount<br/>specialist
    participant P as Poster and<br/>timing<br/>specialist
    participant N as Narration<br/>specialist
    participant S as Run store<br/>(case file)
    participant W as Writer<br/>agent
    participant V as Verifier<br/>(code)

    Note over R: from ④ Rank:<br/>entry flagged<br/>on 3 criteria
    R->>G: suggestion
    Note over G: adds required<br/>specialists
    G->>S: open case file + evidence pack
    Note over A,N: each reads the evidence pack,<br/>may run catalogue queries,<br/>calls the in-region model
    par in parallel
        G->>A: round sum
        A->>S: finding
    and
        G->>P: poster
        P->>S: finding
    and
        G->>N: narration
        N->>S: finding
    end
    opt a finding needs another specialist
        G->>N: one follow-up
        N->>S: finding
    end
    S->>W: findings
    Note over W: fixed order,<br/>disagreements<br/>side by side
    W->>V: draft
    Note over V: pass: to the<br/>⑥ working paper<br/>fail: template
```

- **Same parts as the architecture:** the router suggests, the routing guard (a code node in the LangGraph orchestrator, ADR-017) dispatches, and every routing decision and finding is stored under a hash of its input and replayed on re-runs (ADR-010).
- **Shared case file:** code opens it with an evidence pack of the entry, criteria, calculations and standard read-only queries. Every specialist starts from that pack, may run a few typed catalogue queries (ADR-008), and writes a finding with evidence IDs. Specialists never talk to each other.
- **One follow-up at most:** if a finding needs another specialist's check, the routing guard sends one follow-up inside the case's model allowance. There is no open-ended agent conversation.
- **Disagreement is shown, not settled:** the writer puts conflicting findings side by side and marks the entry for the reviewer. No agent picks a winner, and the ranking doesn't change.
- **Related entries share a case:** reversal pairs, amounts split under an approval limit, and one user's entries in one period are grouped by code before routing, then investigated once.
- **Fixed allowance:** each case's model allowance covers its worst case (router, all three specialists, one follow-up and the writer) and is set before the run.
