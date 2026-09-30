# Tickmark C4: cloud deployment

Where the [containers](c4-containers.md) run in the cloud: two AWS regions in Japan, Tokyo primary and Osaka standby (ADR-015). Every merge into `main` deploys here (ADR-019). The platform choices stay proposals until the load test confirms them (ADR-016).

```mermaid
flowchart TB
    team(["Engagement team"])
    dns["Tickmark address<br/>[DNS name]<br/>points at Tokyo; moved<br/>to Osaka on failover"]
    gh["GitHub Actions<br/>[CI/CD]<br/>on merge to main:<br/>build, then deploy<br/>to both regions"]

    subgraph tokyo[" "]
        tokyoTag["TOKYO · PRIMARY<br/>ap-northeast-1"]
        ecsT["Amazon ECS on Fargate<br/>Run API (rolling update)<br/>Run worker, one per run"]
        rdsT[("Amazon RDS<br/>Run store: database")]
        s3T[("Amazon S3<br/>Run store: files")]
    end

    bedrock["Amazon Bedrock<br/>Claude, Japan profile:<br/>Tokyo or Osaka"]

    subgraph osaka[" "]
        osakaTag["OSAKA · STANDBY<br/>ap-northeast-3"]
        rdsO[("Amazon RDS<br/>read replica")]
        s3O[("Amazon S3<br/>replica")]
        ecsO["Amazon ECS on Fargate<br/>Run API and workers,<br/>scaled to zero"]
    end

    gh -->|"deploy"| ecsT
    gh -.->|"same build,<br/>kept at zero"| ecsO
    team --> dns --> ecsT
    ecsT --> rdsT
    ecsT --> s3T
    ecsT -->|"agent calls"| bedrock
    rdsT -.->|"replication"| rdsO
    s3T -.->|"replication"| s3O

    classDef tag fill:none,stroke:none
    classDef external stroke-dasharray: 5 5
    class tokyoTag,osakaTag tag
    class bedrock,gh external
    style osaka stroke-dasharray: 5 5
```

- **Normal running:** everything runs in Tokyo. Osaka keeps replicas of the database and files, and its Run API and workers stay at zero until a failover (ADR-015, ADR-016).
- **Failover:** promote the Osaka database replica, start the Run API and workers there, and move the DNS name to Osaka. Osaka already has the latest build, so a failover needs no deploy, and each run resumes from its last checkpoint.
- **Deploys:** every merge into `main` goes to both regions. Tokyo runs the new build, and Osaka gets the same build with nothing running, so a failover never starts an older version. GitHub Actions builds the images once, stores them in Amazon ECR (replicated to Osaka) and deploys with a short-lived AWS role. The Run API switches over only after health checks pass, and running workers finish on their own build (ADR-019). While Osaka is in use after a failover, releases go to Osaka only.
- **Model calls** go to Bedrock's Japan profile, which picks Tokyo or Osaka for each call, so a model outage in one region needs no failover (ADR-018).
- **Not shown:** networking (VPC, subnets, load balancer, security groups), the container registry, the telemetry backend in Tokyo, and local development (ADR-021).
