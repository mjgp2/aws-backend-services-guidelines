---
title: Checklist
permalink: /new-service-checklist/
---

Use this checklist when starting a new backend service or reviewing whether one
is ready for first production launch.

## 1. Before You Write Code

- [ ] The service has a service-local tech design with requirements, deployment
      profiles, user stories, architecture, workflow states, and operational
      notes.
- [ ] The service boundary is explicit: one business capability, clear owners,
      and no hidden cross-domain dependencies.
- [ ] If the codebase is a modular monolith, internal domain seams are still
      strict enough that later extraction is an extraction problem, not a
      redesign.
- [ ] The design says which work is synchronous and which work is background.
- [ ] PostgreSQL owns relational state, workflow coordination, uniqueness, and
      idempotency guarantees.
- [ ] S3 owns large object bytes; PostgreSQL keeps only metadata and ownership.
- [ ] Request, response, job, and event contracts are explicit and versioned.
- [ ] Mixed-version rollout expectations are documented for APIs, events, and
      queue payloads.
- [ ] Destructive schema or contract changes are separated from additive rollout
      steps.
- [ ] API authentication and application authorization are treated as separate
      concerns.
- [ ] JWT verification, if used, has explicit issuer, audience, algorithm, key
      cache, and clock-skew defaults.
- [ ] Service-to-service auth uses explicit caller identity, not shared static
      credentials.
- [ ] A basic threat model exists: who the attackers are, what the crown
      jewels are, and the blast radius of a credential leak.
- [ ] The service has zero application-managed AWS access keys by default.
- [ ] If operator access is needed, bastion or shell access is through SSM or an
      equivalent audited path, not open inbound admin ports. If ECS Exec is
      enabled, it is treated as audited exception access rather than normal
      debugging posture.
- [ ] The environment account, VPC CIDR, per-AZ subnet CIDRs, NAT posture,
      interface-endpoint decision, and flow-log decision are chosen explicitly.
- [ ] Service infrastructure, IAM, networking, alarms, queues, buckets, and
      deploy wiring live in IaC rather than ad hoc console state.
- [ ] IaC changes are reviewed with the same discipline as application code,
      with extra care because they can change security boundaries, runtime
      behavior, and cost posture immediately.
- [ ] Storage lifecycle is designed before launch: each object family has an
      owner, retention rule, and cleanup path.
- [ ] The service explicitly chooses direct S3 versus CloudFront for any
      client-facing object path.

## 2. Before First Production Launch

- [ ] One immutable artifact set is built and promoted through environments.
- [ ] Database migrations are part of that artifact set and have a defined
      execution path.
- [ ] Normal rollout is safe across mixed-version API, worker, and consumer
      windows.
- [ ] Runtime rollback is compatible with the newest deployed schema and queued
      messages.
- [ ] Staging qualification, smoke checks, and rollback steps are written down.
- [ ] Hotfix flow still uses immutable artifacts and records what reached
      production.
- [ ] Dashboards and alarms exist for API health, worker health, queue health,
      and database health.
- [ ] Alert routing separates paging alarms from business-hours alarms, for
      example with two SNS topics or channels: one for wake-people-up alerts
      and one for next-business-day alerts.
- [ ] DLQs, queue backlog alarms, and replay or re-drive ownership are defined.
- [ ] Fairness or quota controls are explicit for any shared bottleneck, or the
      decision to defer them is recorded explicitly.
- [ ] Data retention, deletion, backup, restore, RPO, and RTO are documented.
- [ ] Cost guardrails exist: budget alert, anomaly alert, or both.
- [ ] At least one latency SLI and one availability or error SLI have numeric
      targets or alert thresholds.
- [ ] Production desired counts, AZ spread, and autoscaling headroom are sized
      for loss of one AZ, or any reduced-availability posture is an explicit
      decision.
- [ ] A runbook exists for deploy, rollback, and the top failure modes that the
      service is likely to hit.
- [ ] If latency, throughput, backlog recovery, or cost matter materially, a
      representative load or throughput test has been run before launch.
- [ ] The team has explicitly confirmed whether any compliance, legal-hold, or
      WORM retention requirement applies.

## Related Guidance

- [Software](./software/)
- [Infra](./infra/)
- [Networking](./networking/)
- [Ops](./ops/)
- [Database](./database/)
- [S3](./s3/)
- [Auth](./auth/)
- [Delivery](./delivery/)
