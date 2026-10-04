# Tickmark C4: cloud deployment

Where the [containers](c4-containers.md) run in the cloud: two AWS regions in India, Mumbai primary and Hyderabad standby (ADR-015). Every merge into `main` deploys here (ADR-019). The platform choices stay proposals until the load test confirms them (ADR-016).

```mermaid
flowchart TB
    team(["Engagement team"])
    dns["Tickmark address<br/>[DNS name]<br/>points at Mumbai; moved<br/>to Hyderabad on failover"]
    gh["GitHub Actions<br/>[CI/CD]<br/>on merge to main:<br/>build, then deploy<br/>to both regions"]

    subgraph mumbai[" "]
        mumbaiTag["MUMBAI · PRIMARY<br/>ap-south-1"]
        ecsM["Amazon ECS on Fargate<br/>Run API (rolling update)<br/>Run worker, one per run"]
        rdsM[("Amazon RDS<br/>Run store: database")]
        s3M[("Amazon S3<br/>Run store: files")]
    end

    bedrock["Amazon Bedrock<br/>Claude, India profile:<br/>Mumbai or Hyderabad"]

    subgraph hyderabad[" "]
        hyderabadTag["HYDERABAD · STANDBY<br/>ap-south-2"]
        rdsH[("Amazon RDS<br/>read replica")]
        s3H[("Amazon S3<br/>replica")]
        ecsH["Amazon ECS on Fargate<br/>Run API and workers,<br/>scaled to zero"]
    end

    gh -->|"deploy"| ecsM
    gh -.->|"same build,<br/>kept at zero"| ecsH
    team --> dns --> ecsM
    ecsM --> rdsM
    ecsM --> s3M
    ecsM -->|"agent calls"| bedrock
    rdsM -.->|"replication"| rdsH
    s3M -.->|"replication"| s3H

    classDef tag fill:none,stroke:none
    classDef external stroke-dasharray: 5 5
    class mumbaiTag,hyderabadTag tag
    class bedrock,gh external
    style hyderabad stroke-dasharray: 5 5
```

- **Normal running:** everything runs in Mumbai. Hyderabad keeps replicas of the database and files, and its Run API and workers stay at zero until a failover (ADR-015, ADR-016).
- **Failover:** promote the Hyderabad database replica, start the Run API and workers there, and move the DNS name to Hyderabad. Hyderabad already has the latest build, so a failover needs no deploy, and each run resumes from its last checkpoint.
- **Deploys:** every merge into `main` goes to both regions. Mumbai runs the new build, and Hyderabad gets the same build with nothing running, so a failover never starts an older version. GitHub Actions builds the images once, stores them in Amazon ECR (replicated to Hyderabad) and deploys with a short-lived AWS role. The Run API switches over only after health checks pass, and running workers finish on their own build (ADR-019). While Hyderabad is in use after a failover, releases go to Hyderabad only.
- **Model calls** go to Bedrock's India profile, which picks Mumbai or Hyderabad for each call, so a model outage in one region needs no failover (ADR-018).
- **Not shown:** networking (VPC, subnets, load balancer, security groups), the container registry, the telemetry backend in Mumbai, and local development (ADR-021).
