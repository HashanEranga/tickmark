# Instructions for AI coding tools

This file tells AI coding tools how to work in this repository. People follow [CONTRIBUTING.md](CONTRIBUTING.md), and these instructions add to it.

## What Tickmark is

Tickmark ranks journal entries for testing under ISA 240, which Sri Lanka adopts as SLAuS 240. Rules and statistics in code score every entry and choose up to 300. A routed team of agents then explains those entries in a working paper that an engagement reviewer signs. Code decides; agents explain.

## Read before changing anything

1. [specs/00-ground-rules.md](specs/00-ground-rules.md): the rules that hold everywhere, and the glossary.
2. The spec for the component you are changing, in [specs/](specs/).
3. The ADRs that spec cites, in [docs/adr.md](docs/adr.md), for why each decision was made.
4. [docs/scope.md](docs/scope.md), when the question is about what is in or out of scope.

If these documents disagree, or a spec does not answer your question, stop and ask. Do not choose for the team.

## How to work

- Build only components whose spec is Agreed. If a spec is missing, unclear or wrong, propose a change to the spec first, and do not code around it.
- Write the spec's acceptance tests first, using its test IDs, then write the code that makes them pass.
- Cite the requirement IDs you implement in commit messages and pull requests.
- Branch from `develop` as `feature/<short-name>` or `docs/<short-name>`, and open a pull request into `develop`. Never push to `main` or `develop`.
- Make one change per pull request, and keep it small.
- Disclose AI assistance in the pull request description.

## Never

- Use real client ledgers or personal data. Every ledger is synthetic.
- Commit ledgers or their labels, secrets, credentials, `.env` files or the PDFs in `docs/`.
- Name the client firm from the brief anywhere in the repository. The brief is confidential.
- Let a model output change a flag, a score or the ranking (GR-01).
- Use floating-point numbers for money or scores, or let the clock or unseeded randomness affect a decision (GR-05, GR-07).
- Build open-ended agent loops, prebuilt tool-calling agents that decide when to stop, or agent-to-agent conversation (GR-03).
- Give agents SQL, write access or network access (GR-17).
- Pass narration or other ledger text to a model as instructions. It is quoted data (GR-16).
- Send ledger text outside India. That rules out hosted tracing such as LangSmith, and Bedrock's global endpoint (GR-22, GR-23).

## Writing

- Use plain English with British spelling, short sentences, and the glossary's terms with the meanings given there.
- Explain why in docs and commit messages, not only what changed.
- Draw diagrams as Mermaid flowcharts that reuse the box names in [docs/diagrams/architecture.md](docs/diagrams/architecture.md).

## Commands

Build and test commands will be added with the project skeleton. Until then, this repository holds documents only.
