# AWS Backend Service Design Guidelines

These are my opinionated AWS backend service guidelines based on what I've seen
work since 2003 building software for startups, including building systems in
AWS since before there were VPCs.

[Matthew Painter on LinkedIn](https://www.linkedin.com/in/matthewjgpainter/).

## Scope

These docs are for AWS backend services that need clear ownership, predictable
cost, and enough structure to survive startup growth without a large platform
org.

## Document Map

- [Architecture](./architecture.md): runtime, dataflow, and optional analytics
  branches at a glance
- [Networking](./networking.md): VPC, subnet, NAT, endpoint, and bastion shape
  at a glance
- [Delivery](./delivery.md): build, qualification, rollout, rollback, and
  hotfix defaults
- [Monorepo](./monorepo.md): requirements for putting multiple independently
  deployable backend services in one repository
- [Auth](./auth.md): JWT, JWKS, gateway, and service-to-service auth defaults
- [Infra](./infra.md): platform shape, messaging, security, CI/CD posture, cost, and AWS defaults
- [Ops](./ops.md): observability, self-healing, lifecycle, recovery, and operational standards
- [S3](./s3.md): S3 bucket design, storage classes, lifecycle policy, access patterns, and cost controls
- [Software](./software.md): service boundaries, design workflow, request handling, async workflows, contracts, testing, and reliability rules
- [Database](./database.md): Aurora PostgreSQL defaults, schema posture, scaling, and data-retention rules
- [Checklist](./new-service-checklist.md): launch-readiness and adoption
  checklist for new services

## Read This First

- Start with [Checklist](./new-service-checklist.md) for launch readiness.
- Treat the section pages as the canonical rules.
- If a service deviates from a default, record the reason in the service tech
  design rather than letting it drift.
