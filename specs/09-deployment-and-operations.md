# Spec 09 · Deployment and operations

**Status:** Agreed, 7 Oct 2026 · **Owner:** to be assigned · **Prefix:** OPS · **ADRs:** ADR-015, ADR-016, ADR-017, ADR-018, ADR-019, ADR-021 · **Ground rules:** GR-20, GR-22, GR-23, GR-26, GR-29, GR-30

This spec covers where Tickmark runs and how it is kept running:
- the local Docker setup;
- the cloud setup in Mumbai, with a warm standby in Hyderabad;
- the GitHub Actions pipeline;
- telemetry and alarms;
- failover and failback;
- the drills that prove recovery works.

The operations runbook, one of the handbook's six deliverables, is written from §4.3 to §4.5.

## 1. Purpose

Operability carries 10% of the handbook's marks, and the brief's peak season leaves no room for a regional outage to stop engagements (handbook §3, brief §4, ADR-015). The ledger data must also never leave India. This spec makes the environments, the release path and the recovery steps concrete enough to build, test and rehearse.

## 2. Scope

**In scope:**
- The local environment, the cloud resources in both regions, and the infrastructure code.
- The CI/CD workflows, rollback, telemetry, alarms and the residency check.
- Failover, failback and the five drills.

**Out of scope:**
- The application's behaviour (specs 01 to 08).
- Load-test measurements and the cost model's arithmetic (spec 10).

## 3. Requirements

| ID | Requirement | From |
|---|---|---|
| OPS-01 | Tickmark shall start its whole local environment (§4.1) with one `docker compose up` command, without AWS credentials. | ADR-021 |
| OPS-02 | Tickmark's local configuration shall select the `ollama` provider and the pinned Ollama models by name and digest. | ADR-021, GR-20 |
| OPS-03 | Tickmark shall place every cloud resource that stores or processes ledger data in Mumbai (`ap-south-1`) or Hyderabad (`ap-south-2`). | ADR-015, GR-22 |
| OPS-04 | Tickmark shall run the Mumbai resources in §4.2, with continuous replication of the database, the run store and the container images to Hyderabad. | ADR-015, ADR-016 |
| OPS-05 | Tickmark shall keep Hyderabad ready to take over: the same task definitions at zero tasks, its own load balancer, and replicas of the database, run store, images and secrets. | ADR-015 |
| OPS-06 | Tickmark's worker role shall allow Bedrock calls only through the India inference profiles and their models in the two Indian regions. | ADR-018, GR-22 |
| OPS-07 | Tickmark shall use no stored keys: GitHub Actions assumes a short-lived AWS role through OIDC, services use task roles, and the database password lives in Secrets Manager. | ADR-019, GR-26 |
| OPS-08 | Tickmark shall define all cloud resources as Terraform code in the repository, with one module applied to both regions. | ADR-019 |
| OPS-09 | Tickmark shall export OpenTelemetry traces and metrics (§4.3) to CloudWatch and X-Ray in the active Indian region. | ADR-019, GR-22 |
| OPS-10 | Tickmark's telemetry and logs shall carry IDs, counts and hashes only, never ledger text, and CI shall fail if a LangSmith tracing setting appears in any configuration. | GR-23, ADR-017 |
| OPS-11 | Tickmark shall raise the alarms in §4.3 by email to the team, each linking to its runbook section. | ADR-019 |
| OPS-12 | On every pull request, CI shall run the unit tests, the determinism test, linting, a secret scan and the repository content check. | ADR-019, GR-26 |
| OPS-13 | After every merge into `develop`, CI shall send one synthetic case through the agent graph to Claude on Bedrock in Mumbai. | ADR-019 |
| OPS-14 | After every merge into `main`, CI shall build the images once and deploy them by digest. The Mumbai Run API gets a rolling update with ECS's circuit breaker, workers take the new image for new runs only, and Hyderabad gets the same definitions at zero tasks. While Hyderabad is active, releases deploy to Hyderabad only. | ADR-019, GR-30 |
| OPS-15 | After every deploy, CI shall start a check run on the fixture ledger and mark the release failed unless it succeeds within its limits. | ADR-019 |
| OPS-16 | Tickmark shall let a team member roll back by redeploying the previous image digest with one workflow run. | ADR-019 |
| OPS-17 | Tickmark's failover steps (§4.4) shall restore service in Hyderabad within 30 minutes, losing at most about one minute of writes. | ADR-015 |
| OPS-18 | Tickmark's failback steps shall rebuild a replica in Mumbai from Hyderabad and switch back in a quiet window, with no lost runs. | ADR-015 |
| OPS-19 | The team shall rehearse the five drills in §4.5 before the demo, and record each one's timings, data loss, run outcome and cost impact. | Scope §6 |
| OPS-20 | Tickmark shall run a residency check that lists every resource, model ID, telemetry exporter and endpoint, and fails if any would send ledger data outside India. | ADR-015, GR-22 |
| OPS-21 | Tickmark shall tag every cloud resource with its component and region, so that the cost model can split costs. | Scope §6 |

## 4. Data shapes

### 4.1 Local environment

| Service | Runs |
|---|---|
| `api` | The Run API |
| `worker` | One worker, which claims jobs as in the cloud |
| `postgres` | PostgreSQL, with the same schema and row-level security as the cloud |
| `minio` | S3-compatible object storage for the run store |
| `otel-collector` | The OpenTelemetry collector, printing to the console |
| `ollama` | On Linux machines with a GPU; on macOS, Ollama runs natively instead |

The default local model is `llama3.1:8b`, with `qwen3-coder:30b` as an option for better wording. Both are pinned by digest. Local runs use the fixture ledgers.

### 4.2 Cloud resources

| Resource | Mumbai (primary) | Hyderabad (standby) |
|---|---|---|
| Load balancer, HTTPS | Serves the Run API | Ready, with no tasks behind it |
| ECS on Fargate: Run API | At least 2 tasks, with rolling updates | 0 tasks |
| ECS on Fargate: workers | One task per job, up to 12 runs | 0 tasks |
| RDS PostgreSQL | Primary, single Availability Zone to start | Cross-region read replica |
| S3 run store | Versioned, one prefix per engagement | Replica |
| ECR | Images | Replica |
| Secrets Manager | Database password | Replica secret |
| CloudWatch and X-Ray | Telemetry | Used after a failover |
| Route 53 | The API's name points here | Pointed here on failover |

### 4.3 Telemetry and alarms

**Traces:** one per run, with a span for each stage, case and model call. Spans carry run, case and call IDs, counts, token usage, cost and hashes.

**Metrics:** spend per run, calls made and replayed, allowance used, cache hits, queue age, heartbeat age, run duration and replica lag.

**Retention:** 30 days.

| Alarm | Fires when |
|---|---|
| API unhealthy | The health check fails for 2 minutes |
| Worker stalled | A job's heartbeat is older than 2 minutes |
| Queue waiting | A job has waited more than 10 minutes |
| Run at risk | A run is still going 3 hours after it started |
| Model errors | More than 5% of model calls fail within 10 minutes |
| Replica lag | The database replica is more than 60 seconds behind for 5 minutes |
| Replication failing | S3 or image replication reports failures |
| Database strained | CPU is above 80% for 15 minutes, or free storage drops below 20% |

### 4.4 Failover and failback

**Failover**, after the team confirms a Mumbai outage:
1. Promote the Hyderabad database replica.
2. Scale the Hyderabad Run API to 2 tasks, and let workers start there.
3. Point the API's DNS name at the Hyderabad load balancer.
4. Queue interrupted runs again, so that they resume from their checkpoints (spec 08).
5. Switch the deploy workflow to Hyderabad only.
6. Record the outage times and any lost writes.

**Failback**, in a quiet window:
1. Build a new replica in Mumbai from the Hyderabad database, and wait until it has caught up.
2. Stop new runs, and let runs in progress finish.
3. Promote Mumbai, then point DNS and the deploy workflow back at it.
4. Rebuild Hyderabad's replica from Mumbai.

### 4.5 Drills

| Drill | How | Pass |
|---|---|---|
| Worker interruption | Stop a worker task during the agent stage | The run resumes and keeps its decision hash |
| Model-provider outage | Deny the worker role's Bedrock access for 15 minutes | Cases fall back to templates, and the run finishes |
| Database outage | Reboot the primary database during a run | Workers reconnect, and the run finishes with its decision hash |
| Regional failover and failback | Follow §4.4 with a run in progress | Service is back within 30 minutes, at most about one minute of writes is lost, and the run finishes with the same decision hash |
| Deployment rollback | Release a build that fails its health check, then roll back a working one by hand | The circuit breaker restores the last good build, and the manual rollback completes |

## 5. Acceptance tests

| ID | Given | When | Then | Checks |
|---|---|---|---|---|
| OPS-T01 | A fresh clone on a developer machine without AWS credentials | `docker compose up` runs, then a fixture ledger is submitted | Every service starts, and the run finishes with the pinned Ollama model. | OPS-01, OPS-02 |
| OPS-T02 | The Terraform plan for both regions | The residency check runs | Every data resource is in `ap-south-1` or `ap-south-2`, and the check passes. | OPS-03, OPS-08, OPS-20 |
| OPS-T03 | A deployed environment | The resources are listed | Mumbai matches §4.2, Hyderabad holds its replicas and zero tasks, and every resource has its tags. | OPS-04, OPS-05, OPS-21 |
| OPS-T04 | The worker role | It calls a global Bedrock profile, then a Bedrock model in Singapore | Both calls are denied. | OPS-06 |
| OPS-T05 | The repository and the CI configuration | They are searched for keys | No AWS keys or passwords are found, and CI assumes its role through OIDC. | OPS-07 |
| OPS-T06 | A run with marker text in its narration | Its traces and logs are searched | The marker is absent, and every span carries IDs, counts and hashes only. | OPS-09, OPS-10 |
| OPS-T07 | A configuration file that sets LangSmith tracing | CI runs | CI fails. | OPS-10 |
| OPS-T08 | A worker paused past 2 minutes, and a run still going at 3 hours | Each is observed | The matching alarms email the team with runbook links. | OPS-11 |
| OPS-T09 | A pull request that changes a rule and breaks the determinism fixture | CI runs | The determinism test fails, and so does the pull request. | OPS-12 |
| OPS-T10 | A merge into `develop` | CI runs | The Bedrock smoke test sends one synthetic case and reports success. | OPS-13 |
| OPS-T11 | A merge into `main` during a run | CI deploys | The Run API rolls over with no failed status requests, the run finishes on its original image, Hyderabad holds the new definitions at zero tasks, and the fixture check run succeeds. | OPS-14, OPS-15 |
| OPS-T12 | A good release followed by a bad one | A team member runs the rollback workflow | The previous digest is deployed and healthy. *Recovery: rollback* | OPS-16 |
| OPS-T13 | A failover drill with a run in progress | §4.4 is followed | Service is back in Hyderabad within 30 minutes, at most about one minute of writes is lost, and the run finishes. *Recovery: regional failover* | OPS-17 |
| OPS-T14 | Hyderabad active after OPS-T13 | Failback is followed | Mumbai is primary again with no lost runs. | OPS-18 |
| OPS-T15 | The five drills | They are run before the demo | Each has a record with timings, data loss, run outcome and cost impact. *Recovery* | OPS-19 |

## 6. Open questions

None.

**Decided on 7 Oct 2026 (team decisions):**
1. Terraform defines the cloud, with one module applied to both regions.
2. The Mumbai database starts in a single Availability Zone, and an outage of that zone uses the same failover runbook. The cost model shows what a second zone would cost.
3. Telemetry goes to CloudWatch and X-Ray in the active Indian region, and is kept for 30 days.
4. Alarms go to the team by email.
5. After every deploy, a check run on the fixture ledger runs automatically. The releaser still starts one full-size check run by hand (CONTRIBUTING).

---

*Created 2026-10-07.*
