# Tickmark architecture: one batch run

How one engagement's ledger becomes a signed working paper. The circled numbers match the golden-path steps in [scope.md §2](../scope.md), and [entry-investigation.md](entry-investigation.md) zooms into step ⑤ for one entry. The decisions behind each part are in [adr.md](../adr.md).

```mermaid
flowchart TB
    sim["Ledger simulator<br/>400,000 synthetic entries<br/>with seeded anomalies"]
    team(["Engagement team"])
    reviewer(["Engagement reviewer"])

    subgraph region[" "]
        regionTag["TOKYO (PRIMARY)<br/>ledger data stays in Japan"]
        api["Run API<br/>checks access and<br/>starts a versioned run"]
        store[("Run store<br/>ledgers, settings, results;<br/>every run kept")]

        subgraph rules[" "]
            rulesTag["RULES ENGINE (code)<br/>decides what is flagged<br/>and in what order"]
            validate["② Validate<br/>report every rejected row"]
            score["③ Score all 400,000 entries<br/>rules + statistics"]
            rank["④ Rank deterministically<br/>keep up to 300"]
        end

        subgraph agents[" "]
            agentsTag["⑤ ROUTED AGENT TEAM<br/>explains the top 300,<br/>never changes the ranking"]
            router["Router agent<br/>picks the specialists<br/>each case needs"]
            guard["Routing guard (code)<br/>adds required specialists,<br/>caps calls, logs choice"]
            acct["Account and amount<br/>specialist agent"]
            people["Poster and timing<br/>specialist agent"]
            narr["Narration<br/>specialist agent"]
            wri["Writer agent<br/>drafts the justification<br/>from the findings"]
        end

        ver["Verifier (code)<br/>every figure and entry ID<br/>must exist in the evidence"]
        paper["⑥ Working paper<br/>criterion, source record,<br/>calculations, justification"]
        model["Claude on Amazon Bedrock<br/>Japan profile: each call<br/>runs in Tokyo or Osaka"]
    end

    subgraph standby[" "]
        standbyTag["OSAKA (STANDBY)<br/>takes over if Tokyo fails"]
        replica[("Run store replica<br/>copied continuously<br/>from Tokyo")]
        spare["Run API and workers<br/>started only on failover"]
    end

    sim -->|"synthetic ledger"| team
    team -->|"① ledger + settings<br/>⑦ re-run, refined criteria"| api
    api --> validate --> score --> rank
    rank -->|"up to 300 entries,<br/>related ones grouped"| router --> guard
    guard -.-> acct
    guard -.-> people
    guard -.-> narr
    acct --> wri
    people --> wri
    narr --> wri
    wri --> ver
    ver -->|"draft that passes,<br/>or a template"| paper
    paper -->|"to review and sign"| reviewer
    api -.- store
    store -.->|"read-only context"| agents
    agents -.-> model

    team ~~~ regionTag ~~~ api
    api ~~~ rulesTag ~~~ validate
    rank ~~~ agentsTag ~~~ router
    paper ~~~ standbyTag ~~~ replica ~~~ spare

    classDef tag fill:none,stroke:none
    classDef external stroke-dasharray: 5 5
    class regionTag,rulesTag,agentsTag,standbyTag tag
    class model external
    style standby stroke-dasharray: 5 5
```

- **Code decides, agents explain:** rules and statistics choose and rank the entries. The agents only investigate and justify the top 300, and never change the order.
- **Routing:** the router picks specialists for each case, and the guard always adds the one each flagged criterion requires. Dotted arrows run only when routed, and several specialists can work on one case at once; see [entry-investigation.md](entry-investigation.md).
- **Two regions:** Tokyo runs everything. Osaka keeps a replica of the run store, with the Run API and workers switched off; if Tokyo fails, Osaka takes over and each run resumes from its last checkpoint. Every model call runs in Tokyo or Osaka, so ledger data never leaves Japan (ADR-015, ADR-018).
- **⑦ Re-runs:** each one is a new versioned run. The same ledger and settings give the same ranking, and recorded routing decisions and agent outputs are reused rather than paid for again.
- **⑧ Limits:** under 4 hours per run, under USD 40 per engagement, 12 engagements at once in peak season.
- **Where the details live:** the orchestrator is a LangGraph graph built on LangChain (ADR-017). Every step ⑤ box is a node in it and code decides each branch. The routing guard is one of its code nodes and holds each case's model allowance, and each case file, with its evidence pack, lives in the run store. [entry-investigation.md](entry-investigation.md) shows both, using the same boxes as this diagram.
- **Not shown:** evaluation against the seeded anomalies, monitoring, and GitHub CI/CD.
