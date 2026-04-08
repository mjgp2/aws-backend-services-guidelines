---
title: IaC
permalink: /iac/
---

## 1. Purpose

Use this guidance to represent backend infrastructure in code.

[Infra]({{ '/infra/' | relative_url }}) explains the platform defaults. Use
this page to implement and evolve those defaults in an AWS CDK TypeScript
codebase.

The guidance here assumes a TypeScript CDK codebase with shared platform
stacks, service-owned deployment definitions, and thin reusable stack
templates instead of one giant environment file full of service-specific
branches.

## 2. Default Posture

Use TypeScript CDK for all operationally important AWS infrastructure.

- infrastructure that matters to security, delivery, networking, data, alarms,
  or runtime behavior should live in code
- environment shape should be explicit and reviewable
- service-specific infrastructure should stay aligned to the service boundary
- shared platform infrastructure should stay generic and reusable
- console-created snowflakes should be treated as drift to remove, not as a
  normal operating model

The point is not "use CDK because CDK is fashionable." The point is to make
live infrastructure legible, testable, reviewable, and repeatable.

## 3. Recommended Repo Shape

A good CDK repo for this model usually has these layers:

- `bin/`: the CDK app entrypoint
- `lib/config/environments/`: environment-specific config and config types
- `lib/orchestration/`: generic environment composition
- `lib/stacks/`: shared platform stacks by concern, for example networking,
  database, identity, or observability
- `lib/services/`: service-owned infrastructure definitions
- `lib/constructs/`: reusable low-level building blocks
- `lib/template-stacks/`: reusable higher-level stack patterns
- `test/`: stack, construct, and orchestration tests
- `docs/` and `scripts/`: operational notes and operator helpers

That keeps the shape clear:

- shared platform code is not mixed into service-owned runtime stacks
- service-specific code does not leak into the environment orchestrator
- reusable constructs are proved in real service code before they become
  platform-wide defaults

## 4. Architecture Rules For The CDK App

### 4.1 Keep the environment orchestrator generic

The top-level environment stack should compose shared platform stacks and then
delegate service-local infrastructure to service definitions.

- do not add service-specific branches to the environment orchestrator
- discover enabled services from an explicit registry
- build shared platform foundations first
- merge shared outputs back into the environment context before service deploys
- create observability late, after service stacks exist

The environment layer should coordinate. It should not become the home of every
service-specific exception.

### 4.2 Make service-owned infrastructure first-class

Use a service-owned deployment model.

- each deployable service should expose one explicit infrastructure definition
- the definition should decide whether the service is enabled for an
  environment, which runtimes it owns, and which stacks it deploys
- service-local dependencies such as queues, buckets, migration stacks, and
  runtime stacks should live with the service
- service-owned infrastructure should be registered explicitly, not discovered
  by magic filesystem tricks

This keeps repo structure aligned with service ownership.

### 4.3 Make the normal service stack shape explicit

A normal multi-runtime service should usually look something like this:

`lib/services/<service>/`

- `index.ts`
- `shared.ts`
- `service-definition.ts`
- `stacks/`
- `dependencies-stack.ts`
- `migration-stack.ts`
- `api-stack.ts`
- `worker-stack.ts`

Not every service needs every stack, but the default split should be
intentional:

- `dependencies-stack.ts` for service-local queues, buckets, topics,
  subscriptions, and other resources that share the service lifecycle
- `migration-stack.ts` for the one-shot migration task or migration artifact
  wiring when the service owns a database allocation
- `api-stack.ts` for the long-lived ingress runtime
- `worker-stack.ts` for queue-driven or scheduled background runtimes

That split is useful because API, worker, dependencies, and migration usually
have different rollout order, scaling behavior, permissions, alarms, and
failure modes.

Default rules:

- use one runtime stack per deployable runtime class
- do not hide migrations inside the API or worker stack
- do not create unrelated shared resources from a runtime stack
- keep the stack count low, but not by merging together things that need
  different release or operational lifecycles

If a service is simpler than that, fewer stacks are fine. If it is more
complex, add more service-local stacks deliberately. The point is to make the
stack boundaries follow operational reality.

### 4.4 Keep runtime stacks thin

Runtime stacks should mostly be declarative adapters over reusable stack
templates and constructs.

They should usually contribute:

- the runtime spec
- extra environment variables and secret bindings
- queue, bucket, and database bindings
- service-local alarms or IAM that do not belong in the generic layer

They should not reimplement generic ECS, IAM, ALB, logging, or parameter-store
plumbing over and over.

### 4.5 Use reusable templates, but stop before framework mania

Reusable stack templates are useful when they remove repeated platform work
without hiding important behavior.

Good examples:

- a standard ECS runtime stack
- a migration stack pattern
- a release-infra stack pattern
- managed queue, bucket-subscription, or service-registration constructs

Bad examples:

- a generic abstraction created before a second real service proves the pattern
- broad wrappers that make AWS behavior harder to see
- another layer whose only job is to rename CDK concepts into local jargon

## 5. Configuration Rules

Environment config should be explicit, typed, and centralized.

- keep environment-specific values in `lib/config/environments/<env>.ts`
- keep the config shape in one typed contract
- make service definitions and runtime specs the source of truth for what a
  service actually deploys
- prefer explicit registration over hidden discovery
- keep generated parameter names and metadata paths consistent across services

This is the practical model:

- environment files define VPC, ALB, auth, database, and shared defaults
- service definitions declare runtimes and optional extra image registrations
- runtime specs drive the stack inputs for API and worker services

## 6. AWS CDK TypeScript Guidelines

- prefer small, typed helper functions and constructs over giant option bags
- keep CDK code close to AWS concepts; do not invent a private cloud language
- use explicit interfaces for stack props and service contracts
- keep one construct or stack responsible for one main concern
- make dependencies between stacks intentional and visible
- publish important runtime metadata through one standard mechanism such as SSM
- keep naming helpers, IAM helpers, and other shared utilities narrow and
  boring

For TypeScript specifically:

- keep config and service contracts strongly typed
- let compile failures catch broken environment shape or missing stack inputs
- prefer explicit imports and exported interfaces over implicit object shapes
- treat synth-time TypeScript errors as part of infrastructure safety, not as
  optional polish

## 7. Verification Rules

Every infrastructure change should have a repeatable local verification path.

Good defaults:

- install dependencies
- build TypeScript
- lint
- run unit and stack tests
- synth the target environment
- diff the target environment when the change affects live stacks

A well-structured CDK repo should expose named commands for each verification
step so that local development, pre-commit hooks, and CI invoke the same logic.
The exact toolchain (pnpm, npm, make, etc.) matters less than the commands
being named, documented, and consistently runnable from the local machine and
from CI. Good examples: `build`, `test`, `verify:pre-commit`, `verify:ci`,
`cdk synth`, `cdk ls`, and `cdk diff`.

Review IaC changes like code changes, but with a higher bar for care. They can
change live security boundaries, data posture, availability, and cost in one
commit.

## 8. Security And Guardrail Rules

- avoid broad IAM by default; scope permissions to runtime and resource needs
- keep security controls in code rather than relying on remembered console
  posture
- use guardrail tooling such as `cdk-nag`, but do not silence findings casually
- suppressions should be narrow, justified, and reviewed with a concrete
  operational reason
- prefer fixing the underlying resource shape before keeping a suppression

The standard should be: if the platform wants an exception, the code should say
exactly why.

## 9. Ownership Boundary

- platform owns shared environment composition, shared stacks, reusable
  constructs, and the service authoring contract
- service teams own their service definition, service-local dependency stacks,
  migration stack contents, and runtime-specific infrastructure choices inside
  the platform guardrails
- platform should make the common path easy, not become a standing ticket queue
  for routine service infrastructure work

That means a new service should normally be added by creating a service-owned
definition under `lib/services/<service>/`, not by teaching the environment
orchestrator a new hard-coded branch.

## 10. Review Expectations

Changes to the shared IaC repository should receive platform-team review.

- if a pull request changes shared stacks, reusable constructs, environment
  composition, IAM posture, networking, database platform shape, observability,
  or deployment guardrails, platform review should be required
- service teams can still author service-local infrastructure in that repo, but
  changes should be reviewed by the team that owns the shared platform
  standards
- the point of that review is to protect security, scalability, operability,
  and consistency across services, not to turn platform into a slow central
  delivery queue

This is especially important in a shared IaC repository because one apparently
local change can alter the platform contract for many services at once.

## Related Guidance

- [Infra]({{ '/infra/' | relative_url }}): platform defaults, AWS primitives, CI/CD, and security posture
- [Security]({{ '/security/' | relative_url }}): secret scanning, image scanning, and account-level guardrails
- [Networking]({{ '/networking/' | relative_url }}): VPC and ingress shape
- [Delivery]({{ '/delivery/' | relative_url }}): artifact promotion, rollout, and migration ordering
- [Team]({{ '/team-topologies/' | relative_url }}): platform versus product ownership boundaries
- [Repos]({{ '/repos/' | relative_url }}): repository-boundary choices for service ownership
