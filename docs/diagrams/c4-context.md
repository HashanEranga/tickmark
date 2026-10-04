# Tickmark C4: system context

Who uses Tickmark and what it depends on. This is level 1 of the C4 model. The [containers](c4-containers.md) view zooms into Tickmark, and [architecture.md](architecture.md) shows one run step by step.

```mermaid
flowchart TB
    sim["Ledger simulator<br/>[team tool]<br/>makes synthetic ledgers<br/>with planted anomalies"]
    team(["Engagement team<br/>[person]"])
    reviewer(["Engagement reviewer<br/>[person]"])

    subgraph india[" "]
        indiaTag["INDIA<br/>ledger data stays here"]
        tick["Tickmark<br/>[software system]<br/>ranks up to 300<br/>journal lines for<br/>ISA 240 testing and<br/>explains every flag"]
        model["Claude on Amazon Bedrock<br/>[external system]<br/>each call runs in<br/>Mumbai or Hyderabad"]
    end

    sim -->|"synthetic ledger"| team
    team -->|"① ledger + settings<br/>⑦ re-runs"| tick
    tick -->|"case evidence out,<br/>typed findings back"| model
    tick -->|"⑥ working paper"| reviewer

    team ~~~ indiaTag ~~~ tick

    classDef tag fill:none,stroke:none
    classDef external stroke-dasharray: 5 5
    class indiaTag tag
    class model external
```

- **People:** the engagement team submits ledgers and re-runs them; the engagement reviewer signs the working paper. Neither talks to the AI model directly.
- **Claude on Amazon Bedrock** is the only outside system Tickmark calls while it runs. Its India profile keeps every call in Mumbai or Hyderabad (ADR-018).
- **Ledger simulator:** a team tool that feeds the demo and the evaluation. It is not part of the running system, and real client ledgers are out of scope (ADR-004).
- The circled numbers match the golden path in [scope.md §2](../scope.md).
