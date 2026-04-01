---
title: Ops
permalink: /ops/
---

## 1. Purpose

These are the operating defaults I recommend for backend services in
production.

I am not trying to write a heavyweight SRE handbook here. I am trying to set a
practical operating baseline that a small or medium engineering team can
actually maintain.

Delivery and release mechanics live in
[Infra](./infra/). This document focuses on
observability, self-healing, lifecycle, recovery, and operational standards.

## 2. Operational Defaults

### 2.1 Everything observable

Every service must emit:

- structured logs with request IDs, trace IDs, deployment version, and actor or
  tenant identifiers where safe
- RED metrics for request-driven paths: rate, errors, duration
- queue and worker metrics: backlog, age of oldest message, throughput,
  retries, failure rate, concurrency saturation
- database health metrics: connection pressure, query latency, storage growth,
  vacuum or maintenance health where relevant
- health endpoints: at minimum `/livez` and `/readyz`

Tracing default:

- use AWS X-Ray and CloudWatch Application Signals or an equivalent AWS-managed
  path as the default tracing and service-observability posture
- propagate trace IDs across API, worker, queue, and downstream call boundaries
  where practical
- ensure trace IDs are present in structured logs so operators can move from a
  failing request or job to the corresponding trace quickly
- prefer turn-key managed integrations over bespoke tracing pipelines unless a
  clear product or platform need justifies more custom observability work

Sensitive-data default:

- do not emit secrets, raw credentials, tokens, or high-risk personal data into
  logs, traces, metrics, or event payloads
- log stable identifiers and redacted summaries instead of dumping whole
  request or database payloads
- treat observability systems as production data stores with their own access
  control and retention requirements

Observability is not complete unless dashboards and alarms exist. Emitting data
without alerting on it is only partial instrumentation.

### 2.2 Self-healing first

Infrastructure and runtimes should assume processes will die, tasks will be
rescheduled, and messages will be delivered more than once.

Required posture:

- API and worker tasks must tolerate restarts without manual cleanup
- background handlers must be idempotent or converge safely under duplicate
  delivery
- autoscaling should be based on meaningful signals such as CPU, memory, queue
  backlog, or request pressure
- deployments must support rolling replacement without taking the whole service
  down
- graceful shutdown must stop new work, finish or safely release in-flight
  work, and exit predictably
- dead-letter queues should exist for work that cannot be retried forever

“Self-healing” does not mean “no incidents.” It means the common failure modes
degrade safely and recover without operator heroics.

### 2.3 Define data lifecycle up front

Every meaningful data set needs an explicit lifecycle policy before production
launch.

At minimum, each service should define:

- what data is canonical, derived, temporary, or historical
- the retention period for each class
- the trigger for deletion, expiry, or archival
- the system responsible for enforcing deletion
- whether deletion is hard delete, soft delete, archive, or legal hold
- how downstream indexes, caches, replicas, and object storage are cleaned up

If nobody can answer "when does this data get deleted?" the design is
incomplete.

### 2.4 Fairness and quota controls

- define fairness controls for shared systems, not only raw endpoint limits
- use per-tenant, per-principal, or per-workflow quotas where one actor could
  otherwise saturate shared compute, queue, or storage capacity
- make noisy-tenant protection explicit for multi-tenant APIs and background
  workflows

Launch-readiness acceptance criteria:

- each shared bottleneck has an explicit fairness dimension, for example tenant,
  principal, workflow, queue, or export job
- the service defines what happens on quota breach, for example `429`, defer,
  shed, or operator intervention
- if the team is intentionally shipping without fairness controls, that decision
  is explicit in the tech design rather than accidental omission

### 2.5 Define backup and restore posture up front

Every production service needs explicit recovery expectations, not only
retention settings.

Minimum default posture:

- define service-level RPO and RTO before launch
- document which systems are authoritative, which can be rebuilt, and which
  require restore drills
- test database restore and service bootstrap at least once before declaring a
  service production-ready
- do not assume single-region AWS defaults are sufficient DR planning

Example baseline unless a service has stronger recovery requirements:

- one AWS region per environment, with no cross-region failover by default
- Aurora backup retention enabled by environment config; a reasonable baseline
  is `7` days in `staging` and `14` days in `production`
- deletion protection enabled for shared Aurora clusters
- canonical S3 buckets use versioning when accidental overwrite or deletion
  recovery matters; a good default is versioning plus incomplete multipart
  upload expiry after `7` days

Cross-region replication, warm standby, or multi-region active/active should be
exceptions justified by concrete RPO, RTO, regulatory, or revenue risk.

### 2.6 FinOps needs operational guardrails

Cost control is part of service operations, not only quarterly finance review.

Minimum default posture:

- define a rough expected spend envelope for each production service and major
  shared capability
- configure AWS Budgets, cost anomaly detection, or equivalent alerts for
  meaningful spend deviation
- route cost alerts to the same owning team that receives operational alarms
- use tags and application metadata consistently enough that spend can be
  attributed to a service, environment, or shared capability
- investigate unexpected spend changes with the same seriousness as backlog,
  error-rate, or saturation anomalies

Launch-readiness acceptance criteria:

- each production service has a named spend owner
- the service has an expected spend envelope, even if it is rough at first
- at least one budget or anomaly alert exists and routes to that owner
- spend is attributable to the service and environment through tags or
  equivalent metadata

## 3. Launch-Readiness Requirements

Every production service should have:

- at least one request or job latency SLI and one availability or error SLI,
  each with a numeric target or alert threshold
- documented RPO, RTO, and restore procedure for canonical data
- an owner on the hook for alarms and incidents
- an explicit support model for production alarms, for example business-hours
  support or 24x7 on-call, so paging expectations are not implicit
- dashboards for API health, worker health, queue health, and database health
- centrally recorded service ownership metadata for alerting, inventory, and
  incident response
- a documented retention and deletion policy for major data classes
- a cost guardrail such as a budget alert, anomaly alert, or both, with a clear
  owner
- a deployment runbook
- a rollback plan
- smoke tests for staging and production
- environment parity high enough that staging failures are meaningful

Incident-management baseline:

- a paging path for production-impacting alarms
- separate alert-routing channels, for example two SNS topics: one for
  wake-people-up alerts and one for next-business-day alerts
- severity levels and escalation expectations that are understood by operators
- a named primary responder for production alarms during the service's stated
  support window
- an incident channel or equivalent coordination path for active production
  issues
- a post-incident review for meaningful outages, data-loss events, or repeated
  operational failures

Keep post-incident review lightweight: what happened, impact, root cause,
contributing factors, concrete follow-ups, and an owner for each follow-up.

Severity baseline:

- `sev1`: active major customer impact, data loss, or material revenue risk;
  page immediately and coordinate live
- `sev2`: major degradation with workaround or contained blast radius; page the
  owning team promptly
- `sev3`: non-critical degradation, operational debt, or degraded internal
  tooling; handle in working hours unless the trend worsens
