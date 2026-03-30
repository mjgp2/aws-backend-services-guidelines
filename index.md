---
permalink: /
---

# Backend Service Design Guidelines

These are my opinionated defaults for backend services that run in AWS and need
to stay understandable, operable, and cost-aware as they grow.

They come from what I've seen work since 2003 building software for startups,
including building systems in AWS since before there were VPCs.
[Matthew Painter on LinkedIn](https://www.linkedin.com/in/matthewjgpainter/).

The expensive mistakes are usually API contracts, database schemas, and
high-level architecture. Application code is comparatively easier to fix.

## How To Read These Docs

- `must`, `required`, and explicit launch-readiness lists are the minimum bar
  before production.
- `good default`, `recommended`, and similar wording means the baseline I would
  start with unless the service has a concrete reason to differ.
- [Checklist](./new-service-checklist/) is the launch-readiness bar. The
  section docs explain the defaults and the reasons behind them.
- Infrastructure, networking, IAM, alarms, and other operationally important
  AWS configuration should live in IaC, not in remembered console state.

## Documents

- [Architecture](./architecture/)
- [Networking](./networking/)
- [Delivery](./delivery/)
- [Auth](./auth/)
- [Software](./software/)
- [Infra](./infra/)
- [Ops](./ops/)
- [Database](./database/)
- [S3](./s3/)
- [Checklist](./new-service-checklist/)

## Scope

These guidelines target services that:

- are expected to support at least low-millions monthly active users over time
- run in AWS
- are operated by a small to medium engineering team that needs reliability
  without a large dedicated platform org
- need clear operational ownership, predictable cost, and fast incident
  recovery

## Operating Principle

The default platform model is boring on purpose:

- one AWS account per environment
- stateless API containers for synchronous control-plane work
- private worker containers for background and queue-driven work
- Aurora PostgreSQL for relational coordination state
- SQS for durable asynchronous commands and worker jobs
- SNS for simple event fan-out and notifications
- S3 for large object storage
- CloudFront when objects are materially cacheable or when you specifically
  need edge controls in front of S3
- infrastructure provisioned through shared TypeScript CDK AWS modules,
  templates, or platform stacks
- everything observable, everything restartable, and all background work safe
  under retries
- no long-lived application credentials by default; prefer IAM roles, IAM DB
  auth, and presigned access, using Secrets Manager only when a real secret
  cannot be removed

## How To Use These Docs

- Start with Checklist when creating a service or reviewing launch readiness.
- Start with Software for service boundaries, design method, and testing.
- Use Infra for deployment shape, security defaults, CI/CD, and platform
  choices.
- Use Networking for the VPC, subnet, NAT, endpoint, and bastion model.
- Use Ops for observability, lifecycle, recovery, and operational standards.
- Use Database and S3 when the service needs persistent storage decisions.
- These docs work best when teams also keep a service-local tech design and a
  shared IaC layer instead of rebuilding platform conventions in each repo.
- Use the diagram pages for quick orientation, not as a substitute for the
  written rules.
