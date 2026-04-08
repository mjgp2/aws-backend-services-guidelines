---
title: Arch
permalink: /architecture/
---

This is the runtime and dataflow pattern I would start with for most backend
services. Some branches are optional and should only be enabled when the
service actually needs them.

```mermaid
flowchart TD
    client["Client"]
    waf["AWS WAF / Shield<br/>(shared, optional)"]
    alb["ALB (shared)"]
    api["API runtime<br/>AWS Fargate"]
    worker["Worker runtime<br/>AWS Fargate"]
    outbox[("Transactional outbox")]
    db[("Aurora DB")]
    bucket[("S3 bucket")]
    workq[["SQS work queue"]]
    workqdlq[["Work queue DLQ"]]
    eventsq[["S3 Bucket<br/>Event notifications"]]
    eventsqdlq[["S3 event notifications DLQ"]]
    sns[["SNS topics"]]
    kds[["Kinesis Data Streams"]]
    lake[("S3 data lake")]
    redshift[("Amazon Redshift")]
    cfg["Config / secrets<br/>SSM + Secrets Manager"]
    logs["CloudWatch Logs"]
    metrics["CloudWatch Metrics / dashboards<br/>(EMF + AWS metrics)"]
    traces["Tracing / Application Signals"]
    alerts["Alerting"]

    client -->  waf
    waf --> alb
    alb --> api
    client -. "presigned upload / read" .-> bucket
    api --> db
    api --> outbox
    outbox --> workq
    api --> bucket
    api --> sns
    api --> kds
    bucket --> eventsq
    workq --> worker
    workq --> workqdlq
    eventsq --> worker
    eventsq --> eventsqdlq
    sns --> workq
    worker --> kds
    kds --> lake
    lake -. "optional warehouse load" .-> redshift
    worker --> db
    worker --> bucket
    cfg --> api
    cfg --> worker
    db -. "optional zero-ETL" .-> redshift
    api --> logs
    api --> traces
    worker --> logs
    worker --> traces
    logs --> metrics
    logs --> alerts
    metrics --> alerts
    traces --> alerts
```

## Notes

- The `API -> outbox -> SQS` path is the default reliable async publication
  model when queued work depends on committed database state.
- The point of the outbox is to avoid the classic split-brain failure where the
  database state change commits but the queue publish does not, or vice versa.
- The default pattern is: commit business state and outbox record together,
  then let a separate publisher drain the outbox with retries and alarms.
- Direct client upload and download against object storage should use
  authenticated control-plane decisions plus short-lived signed URLs.
- DLQs are required for durable background processing and should have owners,
  alarms, and runbooks.
- Kinesis, data lake, and Redshift are optional analytics branches, not part of
  the minimum runtime architecture.

## Why This Baseline

- Fargate is the default here because it gives long-lived APIs and workers a
  boring runtime model without the platform weight of EKS.
- Lambda is a good fit for smaller event-driven work when runtime limits, cold
  starts, and packaging constraints still fit the service.
- EKS should only be the default when the team is already committed to running
  Kubernetes for broader reasons than one backend service.
- WAF is optional, but it becomes a good default quickly for public APIs that
  face ordinary internet abuse or credential-stuffing noise.

## Related Guidance

- [Software]({{ '/software/' | relative_url }}): service boundaries, contracts, and
  background workflow design
- [Infra]({{ '/infra/' | relative_url }}): compute, queues, security, and
  delivery mechanics
- [Database]({{ '/database/' | relative_url }}): relational state, outbox posture,
  and migration rules
- [S3]({{ '/s3/' | relative_url }}): object lifecycle, access, and
  cost controls
