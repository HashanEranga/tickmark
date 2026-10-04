# Tickmark C4: containers

The separately running parts of Tickmark, and what each one stores or calls. This is level 2 of the C4 model, zooming into Tickmark from the [context](c4-context.md) view. [architecture.md](architecture.md) shows the same parts step by step, and [c4-deployment.md](c4-deployment.md) shows where they run.

```mermaid
flowchart TB
    team(["Engagement team<br/>[person]"])
    reviewer(["Engagement reviewer<br/>[person]"])

    subgraph tick[" "]
        tickTag["TICKMARK<br/>[software system]"]
        api["Run API<br/>[container: Python]<br/>checks access, starts<br/>versioned runs, serves<br/>working papers"]
        worker["Run worker<br/>[container: Python,<br/>one per run]<br/>② validate · ③ score<br/>④ rank · ⑤ agent team<br/>⑥ verifier + working paper"]
        db[("Run store: database<br/>[PostgreSQL]<br/>runs, job table,<br/>case files, agent<br/>outputs, checkpoints")]
        files[("Run store: files<br/>[object storage]<br/>ledger snapshots,<br/>working papers")]
        otel["Telemetry<br/>[OpenTelemetry]<br/>traces and metrics,<br/>never ledger text"]
    end

    model["Claude on Amazon Bedrock<br/>[external system]<br/>India profile"]

    team -->|"① submits ledger<br/>+ settings"| api
    reviewer -->|"⑥ downloads<br/>working paper"| api
    api -->|"saves snapshot"| files
    api -->|"queues run"| db
    worker -->|"reads snapshot,<br/>writes working paper"| files
    worker -->|"claims run, saves<br/>state and outputs"| db
    worker -->|"agent calls"| model
    worker -.->|"traces"| otel
    api -.->|"traces"| otel

    team ~~~ tickTag ~~~ api
    files ~~~ worker
    db ~~~ worker

    classDef tag fill:none,stroke:none
    classDef external stroke-dasharray: 5 5
    class tickTag tag
    class model external
```

- **Run worker:** one per run, started when a run is queued and stopped when it finishes. It holds the rules engine (DuckDB with versioned SQL criteria), the routed agent team (a LangGraph graph) and the verifier from [architecture.md](architecture.md) (ADR-016, ADR-017).
- **Run store:** the single "Run store" of [architecture.md](architecture.md) is two containers here: a database for state and object storage for ledgers and working papers.
- **Model calls** leave only from the worker, and only to Bedrock's India profile; agents have no other network access (ADR-013).
- **Local development** runs the same containers in Docker Compose, with Ollama models in place of Bedrock (ADR-021).
