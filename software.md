---
title: Software
permalink: /software/
---

## 1. Purpose

These are the software structure defaults I recommend for backend services that
need to survive growth, outages, and team change.

The default target is a service that remains understandable after two years of
feature growth, not just one that is fast to ship in week one.

The expensive mistakes are usually API contracts, database schemas, and
high-level architecture. Application code is comparatively easier to rewrite,
so this guidance is biased toward getting the hard structural decisions right
early.

## 2. Architectural Defaults

### 2.1 Start with flows and service tech design docs

Use this design method as the default, and require each substantial service to
capture its concrete choices in a service-local tech design.

Default design sequence:

- start with user stories, operator needs, and deployment profiles
- write the main happy-path and failure-path flows before naming many domain
  objects
- use sequence diagrams for cross-component interaction and state diagrams for
  lifecycle-heavy entities
- derive domain objects, ownership boundaries, contracts, and storage design
  from those flows
- only then lock in queue shapes, table schemas, and external API surfaces

A good reference service tech design usually follows this shape: requirements,
deployment profiles, glossary, user stories, high-level architecture, then
detailed workflow and state decisions.

This matters even more when teams use AI coding agents. Agents are much more
reliable when the service has a clear written spec for requirements, workflow
flows, state transitions, contracts, and test expectations. If those things
are missing, the agent will fill the gaps with plausible guesses, which is
exactly how architecture drift and subtle behavior bugs get introduced.

If teams are using AI coding agents regularly, also use
[AI Coding]({{ '/ai-coding/' | relative_url }}).

Documentation format and location defaults:

- write service tech docs in Markdown
- use Mermaid for sequence diagrams, state diagrams, stack graphs, and other
  architecture visuals when diagrams add clarity
- keep those docs in the service repo alongside the code, not in a separate
  wiki or slide deck that drifts out of date
- update the docs in the same change when architecture, workflow, or public
  behavior materially changes
- for meaningful service work, keep the main happy path, important failure
  paths, retry behavior, state model, and test shape explicit enough that a new
  engineer or an AI coding agent can implement against the document instead of
  guessing
- identify the critical test shape in the design, not just the code: which
  behaviors need automated proof, which ones need integration or contract
  coverage, and which deployed checks demonstrate that the service still works

Do not create a separate organization-wide “software design process” document
unless the design method grows large enough to need its own maintained
standard. Keep the design method here in the software guidance, while each
service keeps its own concrete design doc.

### 2.2 Prefer clear bounded services over premature microservice sprawl

- Start with a well-structured service boundary around a business capability.
- Prefer a modular monolith or a small number of services before splitting into
  many networked services.
- Even inside a modular monolith, keep domain boundaries strict: separate
  modules, explicit ownership, and narrow internal contracts should make later
  extraction possible without re-discovering the system.
- Split services when ownership, scaling, security, or change cadence clearly
  diverge.

The failure mode to avoid is replacing in-process complexity with distributed
systems complexity before the product earns it.

The opposite failure mode is also common: calling something a modular monolith
while allowing broad cross-domain imports, shared mutable data, and hidden
coupling that make future service extraction more expensive than starting with
clear seams.

### 2.3 Keep repository boundaries aligned with service boundaries

If the system is still a modular monolith, one repo is normal. This guidance
applies once you have chosen a deployable service boundary that is expected to
move independently.

Default to one repository per deployable domain service.

That repository should usually contain the parts of the service that share one
delivery lifecycle, for example:

- API runtimes
- worker runtimes
- queue consumers
- service contracts and schemas
- service tests
- service-local infrastructure definitions
- service tech design and runbook docs

Do not split one service into multiple repos per runtime by default if those
parts are developed, tested, released, and rolled back together.

That does not mean one service has to be one package. A service repo can still
use multiple internal packages, modules, or workspaces when that helps keep the
codebase structured, as long as those packages still serve one deployable
service boundary.

Also do not put multiple unrelated domain services in one repo by default and
then try to recreate isolation later in CI, release orchestration, or approval
flows. If services need independent ownership, testing, release cadence, and
rollback, the repo boundary should normally reflect that.

For the full repo-boundary decision, including when a multi-service monorepo is
reasonable and what bar it has to meet, use
[Repos]({{ '/repos/' | relative_url }}).

Shared code is not a reason to collapse unrelated services into one repo. If a
library is genuinely shared and has its own lifecycle, publish it as a shared
package or keep it in a separately owned shared-code repo or shared-code
monorepo.

A well-structured service repo should follow this model: the API, worker,
contracts, service-local docs, tests, and service-specific runtime code all
live together because they form one domain service with one release boundary.

### 2.4 Keep layers explicit

The default service layering is:

- transport layer: HTTP, gRPC, queue consumer, or scheduled entrypoint
- application or service layer: orchestration and use-case logic
- domain layer: business rules and state transitions
- adapters: database, queues, object storage, third-party APIs

This is broadly a hexagonal or ports-and-adapters model: business logic stays
at the center, while transport and infrastructure concerns sit at the edge.

Rules:

- transport code should be thin
- business rules should not live inside controllers or handlers
- persistence details should stay inside repositories or adapters
- external API clients should be isolated behind named interfaces

### 2.5 Make contracts explicit

- Define request, response, and event contracts as first-class code, not
  scattered conventions.
- Validate inputs at the boundary.
- Generate or publish machine-readable API descriptions where practical.
- Version breaking changes intentionally.

If a contract matters to another team or another process, it deserves an
explicit schema.

For asynchronous messaging:

- commands and jobs should have explicit queue payload contracts
- published events should have explicit event schemas and versions
- include an explicit event or job type and schema version in the payload or
  envelope when messages may be processed across mixed-version deploy windows
- event names should describe facts that happened, not instructions to another
  service

Compatibility defaults:

- additive, backward-compatible changes should be the normal contract evolution
  path
- producers should support mixed-version consumers during rollout windows
- same-commit integration tests are not sufficient proof of compatibility when
  independently deployed services or consumers can run different versions of
  the contract at the same time
- consumers should fail deterministically and preserve the message for retry or
  DLQ handling if they receive a version they cannot safely process; do not
  partially apply side effects or silently drop incompatible messages
- breaking changes need an explicit deprecation path, migration plan, and owner
- do not remove or repurpose fields, event meanings, or status values without a
  documented compatibility window

### 2.6 Keep shared server libraries narrow and boring

Shared server libraries are useful, but they should solve proven cross-service
problems rather than becoming a hidden platform framework.

Good candidates for shared libraries:

- configuration loading and validation
- authentication and authorization helpers
- logging, tracing, and metrics wiring
- queue and worker primitives
- database runtime helpers
- HTTP or event contract packages
- small testing helpers for common infrastructure seams

Bad candidates for shared libraries:

- service-specific business logic
- broad internal frameworks that dictate application structure
- abstractions created before two real services need them
- convenience wrappers that hide important database, queue, or network behavior

Rules:

- prefer narrow, stable libraries with clear ownership
- if a library is published or versioned independently, use semver and treat
  compatibility changes honestly
- if a library is only consumed inside one repo, you still need an explicit
  compatibility and upgrade policy even if you are not publishing semver
  releases
- keep service business rules local to the service repo
- extract a shared library only after duplication proves the abstraction, not
  before
- if a shared library becomes hard to upgrade or starts forcing one service's
  design onto others, shrink it or move the logic back into services

The default should be: shared foundations for cross-cutting concerns, local
code for service behavior.

### 2.7 Choose dependencies conservatively

Third-party libraries are part of your architecture. Treat dependency selection
as an engineering decision, not just an implementation detail.

Prefer dependencies that are:

- widely used enough that operational problems and upgrade paths are known
- actively maintained
- well documented
- compatible with the service's language, runtime, and framework posture
- licensed in a way that is acceptable for the organization

Avoid dependencies that are:

- niche and weakly maintained
- effectively abandoned
- difficult to upgrade safely
- broad convenience frameworks that replace understanding with magic
- introduced only because they are trendy or interesting

Rules:

- prefer the most boring dependency that solves the real problem well
- record why a non-obvious dependency was chosen
- prefer shared internal foundations only when they are clearly better than the
  external alternatives they replace
- remove dependencies that no longer justify their operational or upgrade cost

The bar for adding a new dependency should be higher for security-critical,
data-critical, and long-lived backend services than for short-lived internal
tools.

## 3. Request and Workflow Design

### 3.1 Keep synchronous paths bounded

Synchronous APIs should do work that is:

- required for the caller to proceed
- short enough to fit the latency budget
- safe to retry at the request level

Push to async when work is slow, bursty, failure-prone, or operationally
expensive.

### 3.2 Prefer asynchronous workflows for long-running work

Use background workers for:

- data imports
- fan-out jobs
- media or document processing
- third-party synchronization
- reconciliation and cleanup loops

Async workflow rules:

- model durable workflow state explicitly
- assume retries and duplicate delivery
- prefer saga-style multi-step convergence over fake distributed transactions
- expose progress or status when user-facing latency matters

### 3.3 Idempotency is a design requirement

At scale, retries are normal.

Services should support idempotency for:

- externally retried create or mutation calls where duplicate side effects are
  expensive
- queue handlers
- reconciliation jobs
- webhook consumers

Typical tools:

- idempotency keys
- natural uniqueness constraints
- transactional outbox writes
- compare-and-set state transitions

Before launch, idempotency cannot be hand-wavy:

- every externally retried mutation, queue handler, webhook consumer, and
  reconciliation loop has a named idempotency strategy
- duplicate delivery converges to one durable outcome or an explicit no-op,
  rather than duplicating side effects
- the external side-effect boundary is clear: either the side effect happens
  after the dedupe or state-transition guard, or it is itself safe under retry
- critical duplicate and crash-retry paths have automated tests, not just happy
  path tests

### 3.4 Design state transitions, not just CRUD endpoints

For meaningful workflows, define:

- valid states
- valid transitions
- transition owners
- retry behavior
- terminal failure behavior

This is more reliable than letting any code path mutate any status field.

## 4. Reliability Rules

### 4.1 Timeouts and backpressure are mandatory

- Every outbound network call needs a timeout.
- Every service needs concurrency limits somewhere in the stack.
- Queue consumers need bounded worker concurrency.
- APIs should reject or shed load before collapse.

An unbounded queue consumer or unbounded outbound concurrency is a production
incident waiting to happen.

### 4.2 Assume partial failure

- Downstream systems will time out.
- A database commit can succeed after the caller disconnects.
- A queue message can be delivered twice.
- A worker can die after producing an external side effect but before recording
  completion.

Code should converge safely under these conditions.

### 4.3 Prefer explicit configuration and fail-fast startup

- Parse and validate configuration once at startup.
- Keep runtime configuration typed and named.
- Refuse to boot when required config is invalid or missing.

Do not accept silent fallback behavior for production-critical settings.

## 5. API Guidelines

- The concrete JWT and JWKS operational default lives in [Auth]({{ '/auth/' | relative_url }}).
- Use authenticated APIs by default.
- Separate authentication from authorization.
- Authenticate at the boundary, then authorize in application or domain logic
  against an explicit principal, tenant, or ownership record.
- For service-to-service calls, prefer explicit caller identity over shared
  secrets or implicit network trust.
- In AWS, a good default split is: use IAM roles to identify callers to AWS
  managed resources, and use an explicit application-layer auth mechanism for
  direct service-to-service HTTP; then perform application-level authorization
  checks when the callee still needs to decide whether that caller may act on a
  given tenant, resource, or workflow.
- If synchronous service-to-service calls are genuinely required on ECS, prefer
  Service Connect as the baseline connectivity model rather than hand-rolled
  discovery and endpoint wiring.
- Prefer standard JWT verification against an issuer and JWKS-style public key
  set rather than inventing service-specific token formats.
- Cache public signing keys aggressively in API runtimes or API gateways; do
  not fetch the key set from origin on every request.
- If you self-host public verification keys, publish them through a stable
  low-cost static endpoint such as S3, with versioned rotation and overlap during key changes.
- Do not treat knowledge of an object hash, object key, or other opaque
  identifier as authorization by itself.
- Propagate request IDs through logs and traces.
- Return structured error bodies with stable machine-readable codes.
- Version public APIs intentionally.
- Use cursor pagination for high-scale list endpoints.
- Keep response payloads bounded and predictable.
- Separate control-plane metadata from large binary content delivery.

## 6. Background Processing Guidelines

- Queue ownership should usually follow the service or domain boundary that
  owns the workflow.
- Do not make one generic shared queue a dumping ground for unrelated domain
  work.
- Multiple producers are acceptable only when one owning domain defines the
  queue contract, idempotency model, and operational runbook.
- One service may own multiple queues when different workflows need different
  retry, ordering, throughput, retention, DLQ behavior, or SLOs.
- Split worker classes, and usually queues with them, when jobs need materially
  different concurrency, timeout, dependency, permission, or failure-isolation
  characteristics. If jobs have different SLOs, splitting the queues is
  required.
- Workers should own queue consumption and scheduled maintenance loops.
- Publishing a background task that depends on committed DB state should use an
  outbox or equivalent reliability mechanism.
- If a state change and async publication must succeed or fail together, write
  the outbox record in the same database transaction as the state change and
  publish from that durable record afterward.
- If the runtime wakes the publisher immediately after commit, treat that as a
  latency optimization only. Recovery still depends on the durable outbox path.
- Long-running handlers should record progress at durable boundaries, not only
  at the end.
- Retries should distinguish transient from permanent failures.
- Poison messages should land in a DLQ with an owner and a runbook.

For event publication:

- publish events after the source-of-truth state change is durable
- keep events small and stable
- make event type and schema version explicit when events may outlive one deploy
  or be handled by consumers on different rollout versions
- assume subscribers may receive the same event more than once
- do not rely on subscriber execution order unless the transport explicitly
  guarantees it and the design still needs it

Normal rolling deploys should not depend on consumers rejecting messages from a
new producer version. The safer default is compatible evolution across the
mixed-version window; explicit rejection should be the fail-safe for an unsafe
message, not the primary deployment strategy.

Replay and backfill defaults:

- design important background workflows so they can be replayed or backfilled
  intentionally, not only retried one message at a time
- do not treat the DLQ as the long-term replay system for large-scale recovery
  or historical reprocessing
- keep replay and backfill tools idempotent, bounded, observable, and operator
  owned
- if a workflow matters enough to require historical reprocessing, document the
  replay source of truth and the safe re-drive procedure in the service runbook

## 7. Codebase Design Guidelines

- Organize modules by business domain, not by framework primitive alone.
- Keep testable business logic outside transport handlers.
- Avoid shared “utils” packages that become unowned dumping grounds.
- Prefer a small number of well-maintained shared libraries for cross-cutting
  concerns such as config, auth, metrics, and queue helpers.
- Keep generated clients, schemas, and migrations in version control when they
  are part of the shipped contract.
- Keep the migration toolchain and migration entrypoint explicit in the repo so
  CI and release automation can build and run a dedicated migration artifact
  when the service needs one.

Recommended service-repo layout:

- keep service-local docs in the repo under a predictable docs area
- keep the main tech design, operations guide, release runbook, and testing
  docs as first-class maintained documents
- keep API reference material generated from code when practical, but do not
  let generated docs replace hand-written architecture and operational docs

For services with meaningful workflow complexity, a good docs set often
includes:

- `docs/tech-design.md`
- `docs/operations.md`
- `docs/release-runbook.md`
- `docs/testing/coverage-matrix.md`
- `docs/testing/smoke-testing.md`

## 8. Testing

Use [Testing]({{ '/testing/' | relative_url }}) for:

- unit, integration, functional, contract, and smoke-test expectations
- local and CI test posture
- coverage expectations
- workflow coverage matrices
- performance and capacity validation

## 9. Default Platform Decisions

These decisions keep the rest of the guidance coherent and should be treated as
the baseline unless a specific service has a documented reason to differ.

### 9.1 Start with a modular monolith unless there is a clear service split

- Start with a modular monolith when the product is early, the team is small,
  and the deploy boundary does not yet need to split.
- Keep the modular monolith internally partitioned by domain so each module has
  clear ownership, explicit interfaces, and minimal reach into other modules.
- Treat those internal domain seams as real boundaries: avoid shared
  grab-bag packages, cross-domain table access, and convenience calls that
  bypass the owning module's API.
- Split into separate services when ownership, scaling, security, or release
  cadence clearly diverge.
- Do not jump to many networked services just to look “microservice-ready.”

### 9.2 Use REST by default; add gRPC or GraphQL only for clear reasons

- Use REST for externally consumed services and ordinary backend APIs.
- If services need synchronous HTTP communication inside one ECS environment,
  Service Connect is a good default transport and naming layer.
- Add gRPC only when there is a specific internal need such as streaming,
  stricter service-to-service contracts, or performance-sensitive internal
  chatter; gRPC on ALB requires an HTTPS listener, the target group protocol
  version set to `gRPC`, and HTTP/2 backends — this is not the default ALB
  configuration and requires explicit setup.
- Consider GraphQL for BFF (Backend for Frontend) layers and API aggregation
  use cases where clients need flexible query shapes over multiple data
  sources; AWS AppSync is the managed GraphQL option; self-hosted GraphQL
  on ECS is also common.
- Do not force mixed transports onto every service without a concrete benefit.

### 9.3 Prefer templates for scaffolding and libraries for stable shared runtime behavior

- Use templates or scaffolding for repeated repo setup, CI wiring, boilerplate
  docs, and project structure.
- Use shared libraries only for narrow runtime concerns that genuinely benefit
  from one maintained implementation.
- Keep business logic local to the service, not in shared packages.
