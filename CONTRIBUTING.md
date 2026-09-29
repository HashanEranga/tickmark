# Contributing to Tickmark

How the team works in this repository. The reasons are in the [ADR](docs/adr.md): branches and deployment in ADR-019, local development in ADR-021.

## Branches

| Branch | What it holds | Where it runs |
|---|---|---|
| `main` | Exactly what runs in the cloud. Changes arrive only by pull request from `develop` or a `hotfix/` branch. | AWS Tokyo, with the Osaka standby; every merge deploys |
| `develop` | The integration branch, and GitHub's default branch. Every change lands here first. | Team members' machines, in Docker |
| `feature/<short-name>` or `docs/<short-name>` | One change, branched from `develop` and deleted after merging. | Team members' machines |
| `hotfix/<short-name>` | An urgent fix, branched from `main` and deleted after merging. | – |

## Everyday work

1. Start from the latest `develop`:

   ```bash
   git switch develop && git pull && git switch -c feature/<short-name>
   ```

2. Make small, focused commits. Use synthetic data only, and never commit ledgers, secrets or the client PDFs; `.gitignore` already excludes `.env` files and `docs/*.pdf`.
3. Open a pull request into `develop`. CI runs the unit tests and the determinism test.
4. Another team member reviews it. Merge once CI passes and the review is approved, then delete the branch.
5. After the merge, CI sends one synthetic case to Claude on Bedrock as a smoke test. Fix any failure before the next release.

## Releases

1. Open a pull request from `develop` into `main`, titled `Release YYYY-MM-DD`.
2. Before merging, check that:
   - CI and the Bedrock smoke test pass on `develop`;
   - the ADR, specs and diagrams match any design change;
   - the runbook covers any operational change.
3. Merge. GitHub Actions builds the images and rolls them out to both regions. The Run API switches to the new build only after its health checks pass, and a failed rollout rolls back by itself. Runs already in progress finish on the build they started with.
4. Start one check run on a synthetic ledger and confirm it finishes inside the time and cost limits. If the release misbehaves, follow the rollback steps in the runbook.

## Urgent fixes

1. Branch `hotfix/<short-name>` from `main`.
2. Open a pull request into `main`; merging it deploys the fix.
3. Merge `main` back into `develop` straight away, so the fix is not lost.

## Pull requests and commits

- One change per pull request, describing what changed and why.
- Link the spec or ADR it implements, and update the ADR when a decision changes.
- Write commit messages as a short imperative subject line, then a body explaining why.
- Disclose AI assistance in the pull request description (handbook §6).

## Local development

Local runs use Docker Compose, with open-weights models through Ollama instead of Claude (ADR-021). Local results are for building and debugging only; agent quality is measured on Bedrock. Step-by-step setup instructions will arrive with the project skeleton.

## Rules GitHub cannot enforce

The repository is private on GitHub Free, which cannot enforce branch protection. Until the team upgrades to GitHub Pro or Team, these rules hold by agreement:
- nobody pushes directly to `main` or `develop`;
- every change goes through a reviewed pull request.
