---
title: Team Topologies
permalink: /team-topologies/
---

## 1. Purpose

Service design and team design are linked. API ownership, deployability,
schema evolution, incident response, and cost control all get worse when the
ownership model is vague.

The default shape in these docs is product engineering plus platform/cloud.
That maps cleanly to the Team Topologies model: stream-aligned product teams,
a platform team, and occasional enabling or specialist help when needed.

AWS points to Team Topologies directly in both its DevOps guidance and its
internal developer platform guidance. That is a good fit for the posture these
docs are trying to create: safe self-service, lower cognitive load, and clear
ownership.

## 2. Default Team Shape

### 2.1 Product engineering teams are the default stream-aligned teams

- own features end to end
- own the workloads that implement product behavior
- own service-local APIs, queues, schemas, alarms, and runbooks
- ship routine feature changes to production without waiting on a central cloud
  queue
- stay accountable for cost, reliability, and rollback of their services

Product ownership is about product behavior, not where the code happens to
run. If a workload exists to implement product or BFF logic, it belongs with
the product team whether it runs in ECS, Lambda, or Cloudflare Workers.

### 2.2 Platform/cloud owns the paved road

- shared CI/CD patterns and golden paths
- reusable infrastructure-as-code building blocks
- IAM, secrets, and deployment-security defaults
- network patterns and ingress or egress guardrails
- observability foundations and operational standards
- security and compliance guardrails
- scaling, reliability, and cost-visibility patterns
- shared runtime or service templates that remove repeated undifferentiated
  work

The platform/cloud team should operate like an internal product team. The goal
is not to create another approval queue. The goal is to make the safe path the
easy path.

### 2.3 Enabling and specialist teams are exceptions, not the steady state

- use enabling help when a product team needs short-lived support adopting a
  new capability or pattern
- use a specialist team when a subsystem needs unusually deep expertise and is
  not just shared platform plumbing
- do not blur boundaries by letting "shared" become an excuse for muddy
  ownership

## 3. Boundary Rules

- Platform owns shared patterns, templates, controls, and reusable services.
- Product teams own the workloads and behavior running on top of that
  platform.
- Platform should get involved mainly for new shared capabilities, exceptions,
  or work outside the supported patterns.
- "Runs in the cloud" does not mean "owned by platform."
- Do not let shared services fill up with product-specific business rules.

This is the main boundary: platform owns the platform; product owns the
product behavior deployed on top of it.

## 4. Practical Examples

- An ECS API service, worker service, or Cloudflare Worker that exists to
  implement product or BFF behavior belongs with product engineering.
- A shared service for upload, canonical storage, deduplication, generic
  transforms, and stable content-addressed retrieval can sit with platform if
  it stays generic and reusable.
- Database schema scalability, migration posture, and expert guardrails can be
  a platform concern, while product teams still evolve their own data models
  within those supported patterns.
- Monorepo versus repo-per-service is mainly a software choice, but platform
  should define the bar it must meet: independently deployable artifacts,
  promotion by artifact, and safe roll-forward or rollback per service with
  correct migration ordering.

## 5. Preferred Interaction Modes

The normal interaction between product teams and platform/cloud should be
self-service or X-as-a-Service:

- product teams consume templates, paved-road infrastructure, and guardrails
  without needing tickets for normal work
- collaboration is for first-of-a-kind capabilities, exceptions, or
  deliberately time-bounded discovery work
- facilitation is for helping teams adopt patterns and then handing ownership
  back

If long-lived collaboration becomes the default, the platform is probably too
immature, too hard to consume, or owning the wrong things.

## 6. How This Relates To These Docs

- [Infra]({{ '/infra/' | relative_url }}), [Networking]({{ '/networking/' | relative_url }}),
  [Delivery]({{ '/delivery/' | relative_url }}), and
  [Ops]({{ '/ops/' | relative_url }}) describe most of the platform/cloud paved
  road.
- [Software]({{ '/software/' | relative_url }}), [Architecture]({{ '/architecture/' | relative_url }}),
  [Database]({{ '/database/' | relative_url }}), and [S3]({{ '/s3/' | relative_url }})
  describe the design bar product teams should meet inside that road.
- [Checklist]({{ '/new-service-checklist/' | relative_url }}) is the minimum
  production bar regardless of which team owns the service.

If you do not yet have a distinct platform/cloud team, assign named owners for
these responsibilities anyway. The need for the work does not disappear just
because the org chart is still small.

## 7. External References

- [Team Topologies key concepts](https://teamtopologies.com/key-concepts)
- [AWS Well-Architected DevOps Guidance: organize teams into distinct topology types](https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/oa.std.1-organize-teams-into-distinct-topology-types-to-optimize-the-value-stream.html)
- [AWS Prescriptive Guidance: principles of building an internal developer platform](https://docs.aws.amazon.com/prescriptive-guidance/latest/internal-developer-platform/principles.html)
