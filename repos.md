---
title: Repos
permalink: /repos/
---

Use this guidance for repository-boundary choices in backend services.

The default recommendation in these docs is one repository per deployable
domain service.

Shared libraries, templates, and platform packages can live together in a
separately owned shared-code repository or shared-code monorepo when that
improves ownership and reuse without collapsing service release boundaries.

Some teams want a multi-service monorepo for atomic cross-service changes,
shared tooling, and one place to discover code, contracts, and infrastructure.

Those benefits are real. They are also not free. In practice, a multi-service
monorepo is only credible if the platform can preserve service-level
ownership, CI, release, rollback, and incident isolation even though the code
shares one version-control root.

The guidance below is meant to help teams evaluate that choice honestly. If a
multi-service repository cannot meet the required bar, it is still behaving
like one coupled release unit and should be planned that way.

## 1. Default Recommendation

Default to one repository per deployable domain service.

That is the cleanest fit when services need:

- independent ownership
- isolated CI and test execution
- isolated rollout and rollback
- isolated hotfix paths
- clear operational accountability under incident pressure

Do not split one service into multiple repos per runtime by default if those
parts are developed, tested, released, and rolled back together.

That does not mean one service has to be one package. One deployable service
can still be internally split into multiple packages, modules, or workspaces
inside its own repo when that improves structure without creating a second
release boundary.

## 2. Baseline Requirements For Any Service Repo

Whatever repository model you choose, the repo should preserve these outcomes:

- a deployable service has a clear ownership boundary
- the repository is not vague about what release unit it contains
- build, test, deploy, and rollback paths are explicit
- service contracts, tests, docs, and service-local infrastructure stay close
  to the code they govern
- operational ownership stays aligned to the service boundary

For a normal service repo, that usually means keeping together:

- API runtimes
- worker runtimes
- queue consumers
- service contracts and schemas
- service tests
- service-local infrastructure definitions
- service tech design and runbook docs

Default rules:

- do not split one service into multiple repos per runtime if those parts are
  developed, tested, released, and rolled back together
- allow internal packages inside one service repo when they still roll up to
  one deployable service boundary
- keep named owners and review routing explicit
- keep the release artifact aligned to the service boundary, not to an
  arbitrary branch or repository-wide bundle
- keep service-local docs and operational materials in the repo rather than in
  a separate drifting wiki
- if the system is still a modular monolith, one repo is normal until a real
  deploy boundary needs to move independently

This guidance is not saying:

- a modular monolith needs to be split across many repos before the deploy
  boundary is real
- repo-wide validation is forbidden
- shared libraries, shared tooling, or platform packages are bad
- cross-service changes should never be atomic

## 3. Shared Libraries And Shared Code

Shared code should not be used as the reason to collapse unrelated services
into one repository.

Recommended default:

- keep each deployable service in its own repo
- keep shared libraries, templates, and platform packages together in a
  separately owned shared-code repo or shared-code monorepo when they have
  shared ownership, tooling, and change-management needs
- publish or version those shared packages intentionally

This is usually a better fit than putting many independently deployable
services into one repo just because they depend on the same foundations.

Use a shared-code monorepo when:

- several shared libraries or templates have the same owners
- coordinated changes across those packages are common enough that one place
  for testing, compatibility checks, release workflows, and review improves
  engineering quality
- the repo is clearly about shared code, not many product services pretending
  to be one platform

Avoid:

- treating shared code as an excuse to centralize unrelated services
- letting one vague `shared` repo fill up with code that has no clear owner
- forcing product service release cadence to follow the lifecycle of internal
  libraries

## 4. Multi-Service Monorepo Guidance

### 4.1 When A Multi-Service Monorepo Is A Reasonable Fit

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

### 4.2 Published Playbook Status

I am not aware of a published, end-to-end playbook that is sufficient on its
own for running multiple independently deployable backend services in one
monorepo.

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

### 4.3 Outcomes The Repository Model Must Preserve

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

These are the same baseline repo outcomes, but they are harder to preserve in a
multi-service monorepo because the repository shape creates more opportunities
for accidental coupling.

### 4.4 Capabilities Needed To Preserve Those Outcomes In A Monorepo

#### 4.4.1 Independent release isolation inside one repo

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

#### 4.4.2 Impact-aware CI and validation are required

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

#### 4.4.3 Change classes must be explicit

The monorepo should distinguish at least these change types:

- single-service change: one service and its service-local assets
- shared-package or platform change: a change that fans out to multiple
  services and therefore needs broader impact analysis
- intentional multi-service change: one logical change that updates several
  services or contracts together

These are different risk profiles. If the repository treats them all the same,
it will usually either under-test shared changes or over-tax isolated ones.

### 4.5 Evidence For A Strong Proposal

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

### 4.6 Common Failure Signals

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

### 4.7 Decision Rule

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
