---
title: AWS Backend Service Design Guidelines
permalink: /
---

These are [my](https://www.linkedin.com/in/matthewjgpainter/) opinionated defaults for backend services that run in AWS and need
to stay understandable, operable, and cost-aware as they grow.

They come from what I've seen work since 2003 building software for startups,
including building systems in AWS since before there were VPCs.

The expensive mistakes are usually boundaries, contracts, schemas, and
architecture. Application code is comparatively easier to fix.

## Scope

These guidelines target services that:

- are expected to support at least low-millions monthly active users over time
- run in AWS
- are operated by a small to medium engineering organization with distinct
  product and platform/cloud responsibilities, even if the platform function is
  still small
- need clear operational ownership, predictable cost, and fast incident
  recovery

## Operating Principle

The default platform model is boring on purpose:

- one AWS account per environment
- stateless API containers for synchronous control-plane work
- private worker containers for background and queue-driven work
- Aurora PostgreSQL for relational source-of-truth and workflow state
- SQS for durable asynchronous commands and worker jobs
- SNS for simple event fan-out and notifications
- S3 for large object storage
- OpenSearch only when search is a real product capability; keep PostgreSQL as
  the system of record
- CloudFront when objects are materially cacheable or when you specifically
  need edge controls in front of S3
- infrastructure provisioned through shared TypeScript CDK AWS modules,
  templates, or platform stacks
- starting a new service should be cheap: use templates for repo and CI setup,
  shared IaC building blocks for infrastructure shape, and narrow shared
  libraries for cross-cutting runtime concerns
- everything observable, everything restartable, and all background work safe
  under retries
- no long-lived application credentials by default; prefer IAM roles, IAM DB
  auth, and presigned access, using Secrets Manager only when a real secret
  cannot be removed

## How To Use These Docs

- Use Team for platform/cloud versus product-team ownership and operating
  boundaries.
- Use AI Coding for repository guidance, spec quality, and human-review
  boundaries when teams use AI coding agents.
- Use Repos when deciding between repo-per-service and a multi-service
  monorepo.
- Use Architecture, Infra, Networking, Database, and S3 for the main technical
  platform decisions.
- Use Delivery, Testing, and Ops for ship, verify, and run guidance.
- Use IaC for AWS CDK TypeScript repo structure, service-owned infrastructure
  definitions, and infrastructure verification expectations.
- Use the diagram pages for quick orientation, not as a substitute for the
  written rules.
