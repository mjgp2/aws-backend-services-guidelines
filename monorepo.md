---
title: Monorepo
permalink: /monorepo/
---

This guidance applies only when you want multiple independently deployable
backend services in one repository.

The default recommendation in these docs is still one repository per
deployable domain service. A multi-service monorepo is a valid exception model
when teams want atomic cross-service changes, shared tooling, and one place to
discover code, contracts, and infrastructure.

Those benefits are real. They are also not free. A monorepo is only a good fit
when the platform can preserve service-level ownership, CI, release, rollback,
and incident isolation even though the code shares one version-control root.

The requirements below are meant to help teams evaluate that honestly. If the
model cannot meet them yet, the repository is probably still behaving like one
coupled release unit and should be planned that way.

## 1. What This Guidance Is And Is Not Saying

This page is not saying:

- a modular monolith needs to be split across many repos before the deploy
  boundary is real
- repo-wide validation is forbidden
- shared libraries, shared tooling, or platform packages are bad
- cross-service changes should never be atomic

This page is trying to help teams ensure that:

- once services need independent ownership and release lifecycles, the repo
  structure must not destroy that independence
- the repository root is not the release artifact; each service still needs its
  own artifact set and rollback path
- a monorepo must make isolated change, isolated testing, and isolated release
  routine enough that engineers trust the model under pressure

## 2. When A Multi-Service Monorepo Is A Reasonable Fit

A monorepo is usually easier to justify when most of these are true:

- services share one primary language or toolchain family
- teams expect regular atomic changes across service contracts, shared runtime
  foundations, or infrastructure modules
- engineers value one place to search code, contracts, migrations, and docs
- the organization is willing to invest in dependency-graph hygiene, ownership
  rules, and CI tooling instead of assuming the repository shape solves those
  problems automatically

If the main argument is only "we have shared code," that is usually too weak.
Shared libraries and templates are often cheaper than shared release
complexity.

## 3. Published Playbook Status

As of April 7, 2026, I have not found a published, end-to-end playbook that I
would treat as sufficient on its own for running multiple independently
deployable backend services in one monorepo.

I can point to partial tooling patterns:

- Rush, Nx, Turborepo, Bazel, and Pants can help with dependency graphs,
  affected CI, ownership boundaries, packaging, and selective validation
- Azure DevOps and GitHub can help with path-based routing, branch policies,
  and reviewer ownership

That is not the same thing as a complete service playbook.

Most of those tools are centered on packages, projects, or build targets. A
backend service can be modeled inside them, but the tool does not magically
define:

- service-specific deploy orchestration
- rollback semantics
- database migration compatibility
- mixed-version rollout rules
- service-local operational ownership

If you know of a published, field-proven pattern that materially satisfies
those requirements end to end, please contact me with the reference:
[Matthew Painter on LinkedIn](https://www.linkedin.com/in/matthewjgpainter/).

## 4. Required Outcomes

A workable monorepo should preserve these outcomes:

- each service can build, test, deploy, and roll back independently
- unrelated services do not get retested, reapproved, or redeployed on the
  critical path by default
- a hotfix for one service does not require bundling unrelated repository
  changes into the release
- the repository is not the release boundary; the service artifact set is the
  release boundary
- every service still has explicit ownership, contracts, alarms, runbooks, and
  operational accountability

If the model cannot make those outcomes routine rather than exceptional, it has
probably not solved the underlying release and ownership problem yet.

## 5. Required Capabilities

### 5.1 Service identity and ownership are required

- each service must have a clear directory, package, or module boundary
- each service must have named owners
- review routing, CODEOWNERS, or equivalent controls must reflect service and
  shared-package boundaries
- decision authority must stay aligned to service boundaries even when one pull
  request touches many areas

### 5.2 Release isolation is required

- every service must produce its own immutable artifact set
- every service must have its own promotion history across environments
- every service must have its own release identity, for example service-scoped
  artifact naming, tags, or metadata, so one repository commit does not become
  the only release coordinate
- every service must be deployable without rebuilding or redeploying unrelated
  services
- every service must have a documented rollback path to a known-good artifact
- artifact rollback must not require reverting later unrelated commits in the
  repository
- database migrations, queue consumers, and API runtimes must stay aligned to
  the service boundary, not to the repo as a whole
- if one service needs an emergency hotfix, operators must be able to promote
  that service alone

Atomic multi-service changes are fine when they are intentional. The point is
that one commit may touch several services, but release and rollback still have
to work per service afterward.

### 5.3 CI isolation is required

- the repository must support service-scoped build and test commands
- CI must know which services and shared packages are affected by a change,
  including transitive impact where practical
- affected services must run their own quality, unit, integration, and release
  checks without forcing the full monorepo onto the critical path every time
- when impact analysis is uncertain, the system must fail conservatively and
  run more checks, not fewer
- one noisy or broken service must not routinely block unrelated services from
  merging or releasing
- shared security, policy, or platform checks may still run repo-wide, but they
  should be clearly separated from service-scoped validation

Repo-wide validation is still useful as a safety net, periodic confidence
check, or higher-friction gate. It should not be the only serious way to know
whether one service change is safe.

A small monorepo may start with conservative repo-wide validation for a while.
If that is the current posture, record it honestly and define the scale or
latency threshold at which service-scoped validation becomes mandatory.

### 5.4 Testing must follow service boundaries

- each service must own a runnable test suite that proves its behavior without
  depending on unrelated services being built first
- shared libraries must have their own tests and compatibility expectations
- integration and functional tests must be attributable to a specific service
  or contract, not to an undifferentiated repo-wide test blob
- contract compatibility across services must be explicit for APIs, events, and
  queue payloads
- testing only "all affected services from the current HEAD commit" is not
  sufficient evidence when services deploy independently; the monorepo must
  prove the relevant mixed-version and rollback combinations as well
- mixed-version rollout and rollback rules still apply; the repo shape does not
  remove compatibility requirements

### 5.5 Dependency hygiene is required

- service directories, packages, or modules must have explicit ownership and
  allowed dependency rules
- cross-service imports must be narrow, intentional, and reviewable
- shared libraries must stay small, boring, and compatible with independent
  service release
- internal shared packages do not need public-package semantics, but they do
  need an explicit compatibility and change-management policy
- if a shared package change can affect many services, the monorepo must know
  which ones and prove them before release

### 5.6 Infra and operational isolation are required

- each service must have its own infrastructure definitions or clearly isolated
  stack inputs
- deploy permissions, runtime metadata, and rollout controls must stay scoped
  per service
- dashboards, alarms, SLOs, DLQs, runbooks, and on-call expectations must stay
  owned per service
- incident response must allow rolling back, pausing, or scaling one service
  without creating collateral release risk for others
- service-local docs must remain close to the code and release artifacts for
  that service

### 5.7 Change classes must be explicit

The monorepo should distinguish at least these change types:

- single-service change: one service and its service-local assets
- shared-package or platform change: a change that fans out to multiple
  services and therefore needs broader impact analysis
- intentional multi-service change: one logical change that updates several
  services or contracts together

These are different risk profiles. If the repository treats them all the same,
it will usually either under-test shared changes or over-tax isolated ones.

## 6. Evidence For A Strong Proposal

It is worth moving beyond "the tooling can do it" and working through a
concrete design together.

A strong proposal should be able to show:

- the service directory or package model and the ownership map
- the dependency graph rules, including how shared libraries are tracked
- the exact impact-analysis logic for deciding which services need CI
- the release identity scheme for each service, including how artifacts are
  named, tagged, or otherwise identified independently of the repo head
- one example change that affects only one service and the resulting CI and
  deploy path
- one example shared-package change and the resulting impact analysis
- one example intentional multi-service contract change and the resulting
  validation path
- one example compatibility gate that tests a new service artifact or contract
  against a previously qualified peer version, rather than only against peers
  built from the same commit
- one example service rollback that repoints to an older service artifact
  without reverting unrelated repo changes
- one example emergency hotfix for a single service during unrelated ongoing
  work in the same repository
- the fallback behavior when impact analysis is ambiguous or tooling is wrong
- the policy for contract compatibility and internal-package change management

When possible, prefer a real rehearsal over a slide deck:

- run or simulate a service-only change through CI, deploy, and rollback
- run or simulate a shared-package change and show the impacted-service fan-out
- show what operators would do when one service must hotfix while another
  service has newer unrelated commits on the branch

If those workflows remain abstract, the proposal is still theoretical and will
benefit from more design work before teams rely on it operationally.

## 7. Common Failure Signals

These are usually signs that the monorepo is recreating coupling instead of
removing it:

- the main branch or repo state is still treated as the release artifact
- rollback requires reverting unrelated changes that happened later in the same
  repository
- most merges trigger full-repo CI because the dependency graph is too muddy to
  trust selective execution
- one shared library or infra package effectively forces lockstep upgrades
  across many services as a normal operating mode
- service-local ownership is weak, so broad repo reviewers become the
  bottleneck
- migrations, contracts, or deploy approval still happen at repo level rather
  than service level
- incident response becomes slower because release state is harder to isolate

## 8. Decision Rule

A multi-service monorepo is a good fit only if the platform can make it behave
like many independent services that happen to share one version-control root.

The monorepo earns its keep when it gives you shared visibility and atomic
change without collapsing service autonomy.

If the repo still behaves like one shared release boundary, the organization is
not really getting the full monorepo upside. It is still carrying much of the
coupling cost without the intended service isolation.

## Related Guidance

- [Software]({{ '/software/' | relative_url }}): service boundaries, shared code,
  and repository defaults
- [Delivery]({{ '/delivery/' | relative_url }}): immutable artifacts, promotion,
  rollback, and mixed-version rollout rules
- [Infra]({{ '/infra/' | relative_url }}): deploy scoping, CI/CD posture, and
  service-local infrastructure ownership
- [Ops]({{ '/ops/' | relative_url }}): runbooks, alarms, ownership, and incident
  response posture
- [Checklist]({{ '/new-service-checklist/' | relative_url }}): launch and
  adoption readiness criteria
