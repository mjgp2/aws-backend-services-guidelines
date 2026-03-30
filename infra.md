---
title: Infra
permalink: /infra/
---

## 1. Purpose

These are the infrastructure defaults I recommend for backend services.

I am not optimizing for maximum theoretical flexibility. I am optimizing for a
platform that a startup team can run safely at scale with a small number of
engineers.

Observability, lifecycle, recovery, and incident posture live in
[Ops](./ops/). This document focuses on platform
shape, AWS primitives, security, delivery, cost, and tagging.

The diagram view of the VPC, subnet, NAT, endpoint, and bastion model lives in
[Networking](./networking/).

## 2. Default Production Shape

The default backend deployment model is:

- one AWS account per environment
- public API service behind an ALB with healthchecks
- private worker service for asynchronous work
- ECS Fargate as the default compute platform
- ECS Service Connect for required synchronous service-to-service calls inside
  the environment
- Aurora PostgreSQL as the default relational datastore
- SQS for queue-driven background execution
- SNS for event fan-out, notifications, and decoupled subscriber delivery
- S3 for object storage and large payloads
- CloudFront in front of S3 when objects are materially cacheable or when you
  specifically need edge controls in front of S3
- SSM Parameter Store for runtime metadata and Secrets Manager only for
  unavoidable secrets
- infrastructure managed through shared TypeScript CDK AWS building blocks

This is the default because it is operationally boring in the right way:
managed services, strong AWS support, and a low amount of undifferentiated
platform work.

Lambda is still a good fit for smaller event-driven work when runtime limits,
packaging, and cold-start posture fit cleanly. The default here assumes
long-lived API and worker runtimes.

The account boundary matters here. A good default is one AWS account per
environment, with the primary service VPC, shared environment resources, and
service runtimes living inside that account.

### 2.1 Network and VPC baseline

Start with an explicit VPC design rather than generated defaults you barely
understand.

- use one AWS account per environment, with the primary service VPC living in
  that account
- keep public subnets for internet-facing edges and private subnets for app
  runtimes, data stores, and operator paths
- do not assign public IPs to long-lived service runtimes
- keep peerings, routes, and cross-network reachability explicit and narrow

Detailed CIDR planning, NAT posture, interface-endpoint decisions, flow-log
tradeoffs, and bastion posture live in [Networking](./networking/).

## 3. Infrastructure Rules

### 3.1 Split synchronous and asynchronous roles

- API containers should handle short-lived request/response work.
- Worker containers should own queue consumption, scheduled jobs, and long or
  failure-prone workflows.
- Do not run API and worker roles in the same long-lived process in production.
- One container should own one main responsibility.

This keeps scaling, rollback, health checks, and failure handling simple.

### 3.1.1 Size runtimes to survive an AZ failure

For staging and production, assume one AZ can disappear at the worst possible
time.

- spread long-lived ECS services across at least two AZs unless the environment
  is explicitly low-risk
- desired count is not just a throughput number; it also has to leave enough
  healthy capacity after losing one AZ
- if a service runs in `2` AZs, `2` tasks is only the bare minimum for one task
  per AZ; it is not enough if one remaining task cannot carry the service
- if a service runs in `3` AZs, `3` tasks is only the bare minimum for one task
  per AZ; production services often need more so the remaining `2` AZs can
  absorb the loss of the third
- worker fleets should use the same logic: if backlog recovery or latency
  objectives matter after an AZ loss, size desired count and concurrency for
  that failure case, not only for happy-path steady state
- subnet spread by itself does not buy resilience; the runtime needs enough
  tasks, healthy-target budget, and autoscaling headroom to survive the AZ loss

The practical rule is simple: capacity should be planned against `N-1` AZs, not
`N` AZs, unless the service explicitly accepts reduced availability during an AZ
event.

### 3.2 Prefer managed services over self-hosted infrastructure

- Prefer Aurora over self-managed PostgreSQL.
- Prefer SQS over self-managed queue brokers unless ordering or throughput
  requirements clearly exceed SQS fit.
- Prefer SNS for simple AWS-native pub-sub instead of bespoke event fan-out
  wiring between services.
- Prefer S3 over storing large objects in the database.
- Prefer CloudFront over direct S3 delivery when objects are materially
  cacheable, or when you specifically need edge controls such as WAF at the
  edge, custom-domain delivery, or hiding the S3 origin behind one public
  entrypoint.

Detailed S3 policy, including bucket layout, storage classes, lifecycle rules,
and signed-access posture, lives in [S3](./s3/).

Teams should only self-host infrastructure when a managed alternative creates a
clear product or cost problem.

### 3.3 Use async messaging deliberately

SQS queues are the default tool for:

- work that exceeds request latency budgets
- work that depends on unreliable external systems
- workflows that benefit from retries and isolation
- fan-out or burst smoothing

Queue design rules:

- keep queue ownership aligned with a single service or domain boundary
- do not make unrelated domains share one queue and then fight over retries,
  DLQs, ordering, and runbooks
- if multiple producers send to one queue, one owning domain must still define
  the payload contract, compatibility rules, and operator runbook
- it is fine for one domain to own multiple queues when workflows need
  different retention, ordering, throughput, retry, or DLQ behavior
- split worker classes, and usually queues with them, when workloads need
  materially different concurrency, timeout, IAM, dependency, cost, or
  failure-isolation characteristics
- assume at-least-once delivery; idempotent consumers, bounded retries, and a
  DLQ are part of the default
- keep messages small and durable; store payloads in S3 if they are large
- publish queue work through an outbox pattern when the enqueue depends on a
  committed database state change

Do not use a queue to hide a badly designed synchronous dependency. If a user
must see the result before the request returns, it is not background work.

SNS topics are the default tool for:

- simple domain-event fan-out to multiple downstream consumers
- notifications where subscribers should react independently
- lightweight AWS-native pub-sub integration

Use SNS when the producer is announcing that something happened. Use SQS when
the producer needs work to be performed.

Detailed contract, compatibility, outbox, and mixed-version messaging rules
live in [Software](./software/) and the runtime flow is illustrated in
[Architecture](./architecture/).

### 3.4 Standardize deployment and provisioning

The diagram view of this release flow lives in [Delivery](./delivery/).

- Everything important should live in infrastructure-as-code. Do not accept
  console-built snowflakes for service infrastructure, IAM, networking, alarms,
  queues, buckets, or deploy wiring.
- All service infrastructure should be created through shared TypeScript CDK
  AWS modules, stacks, or templates.
- Review IaC changes like code changes, but with an even higher bar for care.
  They can change live security boundaries, data exposure, cost, and failure
  modes in one commit.
- The shared layer should own common concerns: VPC wiring, ECS service shape,
  IAM role posture, security controls, and standard deployment scaffolding.
- Service repositories should declare intent and service-specific inputs, not
  rebuild platform scaffolding from scratch.

This reduces configuration drift and prevents each team from inventing a
slightly different production stack.

### 3.4.1 Build once and deploy the same artifact

The release unit should be a tested artifact for one service, not the moving
state of a branch.

Default posture:

- build one deployable artifact per service runtime
- promote that same artifact through environments
- pin deployments to immutable artifacts such as image digests, not just tags
- keep release, rollback, and approval boundaries aligned with the service
  boundary

Avoid:

- rebuilding separately for staging and production
- deploying directly from repository head as if branch state were the release
  artifact
- coupling unrelated services into one deploy batch because they share a repo

If two services need independent rollout and rollback, they should not depend on
one shared branch-level release unit.

### 3.4.2 Deploy through OIDC and runtime metadata, not static credentials

Default CI/CD posture:

- CI systems should assume cloud roles through OIDC or an equivalent
  short-lived federated identity mechanism
- do not give deployment automation long-lived static AWS credentials
- publish runtime artifact locations and deployment metadata through a shared
  metadata system such as SSM Parameter Store

For a shared TypeScript CDK AWS infrastructure layer, a good default deploy
shape is:

1. build and publish one immutable artifact set for the service, including the
   runtime and, where needed, a dedicated migration artifact
2. authenticate to AWS through an OIDC-assumed deployment role
3. run additive database migrations as a one-shot ECS/Fargate task before
   rolling workers or APIs when schema changes are part of the release
4. update the runtime's artifact metadata parameter, for example an SSM
   parameter holding the image URI or digest
5. refresh the runtime stack so the infrastructure layer rolls the service to
   that artifact

This pattern keeps the infrastructure definition stable while letting operators
or automation promote new artifacts safely.

Rollback should use the same mechanism in reverse: point the metadata parameter
back to a known-good artifact and refresh the affected stack.

If migrations are packaged separately, they should still be built, versioned,
qualified, and promoted as part of the same service release set.

For AWS container platforms, the default execution model should be a dedicated
one-shot ECS/Fargate task for the migration artifact, not migration logic
hidden inside API or worker startup. That keeps the step explicit, auditable,
and independently retryable.

When schema changes are involved, the deploy order should normally be:

- additive migration first
- worker or async runtime rollout next where relevant
- API rollout after that

If the migration is not backward-compatible across a mixed-version window, the
design is not ready for a normal deploy path yet.

### 3.4.3 Use staged qualification before production promotion

For services with meaningful production risk, do not treat the first production
deploy as the qualification environment.

Default promotion posture:

- publish one candidate artifact set for the target commit
- deploy that artifact set to staging first
- run smoke tests and other release checks against staging
- record which exact artifact set qualified
- promote that same qualified artifact set to production

This can be implemented with release records, manifests, or equivalent metadata,
but the invariant matters more than the mechanism: production should receive the
same artifact set that already qualified elsewhere.

Good release pipelines keep:

- artifact publication separate from environment promotion
- staging qualification separate from production promotion
- environment-specific credentials and approvals scoped through OIDC-assumed
  roles and CI environment controls
- smoke verification as part of qualification and promotion, not only as a
  manual afterthought

### 3.5 Security defaults

- Keep databases, API tasks, and workers in private subnets.
- Keep ECS runtimes on private networking with no public IPs.
- Give each runtime its own security group where practical rather than one
  shared broad-access group for everything.
- Use IAM task roles rather than static AWS credentials.
- Prefer IAM-authenticated AWS access paths that remove secrets entirely, such
  as S3 access by task role and RDS IAM auth for ECS runtimes.
- ECS Exec is useful for exceptional live inspection of a running container when
  logs, metrics, and traces are not enough. Treat it as an audited break-glass
  operator tool, not the normal way the team learns how the service works.
- If a bastion host is needed for exceptional operator access, allow access only
  through AWS Systems Manager Session Manager or an equivalent audited
  identity-aware path; do not open SSH, RDP, or other inbound admin ports.
- Encrypt data at rest and in transit.
- Terminate public TLS at the ALB or CloudFront boundary, then keep internal
  traffic scoped to the VPC.
- Treat secret rotation and revocation as operational requirements, not future
  work.
- Put rate limits and abuse controls in front of expensive or externally
  reachable endpoints.

Internal trust defaults:

- do not rely on network location alone as proof that an internal caller should
  be trusted
- use explicit caller identity for service-to-service calls and explicit
  resource policies for queues, topics, and buckets
- for required synchronous service-to-service calls on ECS, prefer Service
  Connect over ad hoc service discovery or handwritten internal endpoint wiring
- use IAM roles to identify workloads to AWS-managed resources, but do not
  assume a direct internal HTTP callee can infer caller identity from AWS role
  alone; carry explicit caller identity at the application layer when the
  callee needs to authorize the action
- prefer narrow per-service permissions over broad shared runtime roles
- grant database ingress only to runtimes that actually need database access

Data-handling defaults:

- classify data at least well enough to distinguish public, internal,
  confidential, and regulated or high-risk data
- use customer-managed or centrally managed KMS keys where the platform or
  compliance posture requires key separation or auditability
- define whether regulated data may appear in logs, traces, events, caches, and
  object metadata before launch, not after an incident
- if residency, legal-hold, or compliance constraints apply, capture them in
  the service tech design and storage lifecycle policy explicitly

Configuration boundary defaults:

- use SSM Parameter Store for non-secret runtime metadata and mutable deploy
  selectors such as image refs, hostnames, queue URLs, and health check paths
- use Secrets Manager only for real secrets that cannot be designed away
- treat JWT public verification keys as public metadata, not secrets; if the
  organization self-hosts a JWKS document, a static distribution path such as
  S3 plus CloudFront is a reasonable default
- default new services to zero application-managed AWS access keys
- if a runtime still needs a secret, record the owner, rotation plan, and
  failure mode explicitly

### 3.6 Cost is a first-class architecture concern

At startup scale, cost problems are reliability problems in waiting.

Every service should have explicit guardrails for:

- queue backlog growth
- runaway egress
- unbounded storage retention
- noisy-tenant behavior
- expensive retries against downstream systems
- oversized per-request compute and memory usage

### 3.6.1 Use Fargate Spot deliberately

Fargate Spot is a good default cost optimization for worker-style runtimes and a
conditional optimization for APIs, but it should not be treated as free
capacity with no interruption cost.

Default posture:

- prefer mixed capacity providers rather than Spot-only production services
- keep an on-demand base for services that must stay available during Spot
  interruptions
- use Spot more aggressively for queue workers that can tolerate restart and
  lease handoff
- use Spot more cautiously for ingress APIs, keeping enough on-demand capacity
  to preserve healthy targets during interruption or replacement

Operational requirements when using Fargate Spot:

- deployments should run with `minimumHealthyPercent: 100` and
  `maximumPercent: 200` so replacement tasks can become healthy before old
  tasks are drained
- graceful shutdown and ECS `stopTimeout` must leave enough time to finish or
  safely release in-flight work
- API deregistration delay should be at least as long as API shutdown drain
  delay
- worker handlers must be idempotent and safe under interruption mid-flight

Example deployment pattern:

- capacity-provider strategies may mix `FARGATE` and `FARGATE_SPOT`
- reference services should keep an on-demand base and then burst with Spot
  rather than depending on Spot alone

### 3.7 Let the shared infrastructure layer manage application tags and metadata

Do not ask each service repo to invent its own tagging scheme. The shared
infrastructure layer should apply the platform tagging model consistently.

Default tagging model:

- service-owned stacks are tagged with the AWS-vended `awsApplication` tag from
  Service Catalog AppRegistry
- each service application is registered in AppRegistry with `Application` and
  `Environment` tags, plus `Service` for service-scoped applications
- shared platform capabilities such as shared data infrastructure use
  `Application`, `Environment`, and `Capability`
- service-level objectives and similar observability resources also use
  `Environment`, `Service`, and `SloType` where relevant
- ownership should be recorded centrally in application metadata, alert routing,
  runbooks, or service catalog records rather than invented ad hoc in each
  stack

Tag keys in use:

- `awsApplication`: AWS-vended AppRegistry application identity attached to
  tagged stacks by the shared infrastructure layer
- `Application`: bounded application or shared capability name
- `Environment`: deployment environment such as `staging` or `production`
- `Service`: service name for service-scoped application and observability
  resources
- `Capability`: shared capability name for non-service shared infrastructure,
  for example shared data infrastructure
- `SloType`: observability classification such as availability, latency, or
  queue health on SLO resources

Rules:

- treat these keys and their capitalization as platform-managed, not ad hoc
  per-service choices
- add or change tag keys through the shared infrastructure layer, not one
  service stack at a time
- keep application identity aligned across tags, AppRegistry metadata, and
  OTEL resource attributes such as `service.name` and
  `deployment.environment`
- keep ownership information in one centrally managed metadata system rather
  than scattering near-duplicate `owner` tags with inconsistent values
- if finance, compliance, or inventory tagging expands later, add it centrally
  in the shared infrastructure layer rather than documenting a second parallel
  scheme here

## 4. Reference Deployment

The preferred baseline is:

1. ALB receives public HTTPS traffic.
2. API ECS service handles synchronous control-plane requests.
3. Aurora PostgreSQL stores relational and workflow state.
4. An outbox or equivalent reliable publication path submits async work to SQS.
5. Domain events and notifications may publish to SNS when multiple
   subscribers should react independently.
6. A worker ECS service consumes SQS and performs background processing.
7. S3 stores large objects, exports, and other object payloads.
8. CloudFront fronts S3 when objects are materially cacheable or when those
   edge-control requirements justify the extra complexity.

This baseline should cover most internal APIs, workflow systems, ingestion
services, and asset-processing systems.
