---
title: Testing
permalink: /testing/
---

## 1. Purpose

This page captures the default testing posture for backend services.

Treat it as the testing menu, not a demand that every service carry every test
type on day one. The launch-readiness minimum bar is the smaller set that
protects the real failure modes of the service.

## 2. Testing Expectations

Every service should have:

- unit tests for domain and adapter edge cases
- integration tests for database and external adapter behavior
- functional or end-to-end tests for critical user and operator flows
- contract tests for important public APIs or events
- smoke tests for deployed environments

The test strategy should match the risk of the service. Queue-driven workflows,
payments, and destructive mutations need stronger integration coverage than
simple read APIs.

### 2.1 Minimum launch-readiness bar

- automated tests for critical domain logic and negative paths
- integration coverage for the database, queue, and external adapter behavior
  that can lose data or duplicate side effects
- smoke tests against the deployed service
- contract checks for important public APIs or async payloads

### 2.2 Recommended local test posture

- use fast unit tests as the default inner loop
- use Docker Compose or an equivalent local orchestration layer when
  integration or functional tests need real Postgres, queues, object storage,
  or other stateful dependencies
- keep local integration environments close enough to production behavior that
  failures are meaningful, without reproducing every production concern locally

## 3. CI Expectations

- CI should run named, repeatable commands rather than bespoke shell logic in
  many places
- local hooks such as pre-commit or pre-push should reuse the same named
  verification commands that CI uses where practical
- pre-commit filters should include secret scanning so obvious credential leaks
  are caught before they ever leave a developer machine
- separate fast quality checks from slower integration and functional tiers so
  failures are easier to interpret and rerun
- every pull request should run the fast test layers needed to protect the
  service, at minimum unit tests and the relevant contract checks
- integration tests should run in CI for services where database, queue, or
  external-adapter behavior is part of the risk profile
- functional tests should exist for the highest-value workflows and should run
  in CI or in a gated pre-release path
- smoke tests should validate the deployed artifact in staging and production
- in a multi-service monorepo, each service should expose its own named build,
  unit-test, integration-test, and package commands
- in a multi-service monorepo, CI should support service-scoped validation for
  isolated changes and broader fan-out for shared-package or multi-service
  changes

### 3.1 Recommended CI layers

- quality or static analysis: formatting, linting, typechecking, dependency or
  import-boundary checks, and other non-runtime safety checks
- unit tests with coverage reporting
- integration tests against real backing services where required
- functional tests against a production-like local composition
- environment smoke tests against staged or promoted deployments

## 4. Coverage Expectations

- track test coverage trends over time rather than using one vanity percentage
  as the only quality signal
- use coverage reporting to find untested critical paths, not to justify weak
  tests
- raise the bar on coverage and negative-path testing around workflow logic,
  retries, authz, and destructive mutations
- for pull requests, consider changed-line coverage or diff-aware coverage as a
  stronger signal than repo-wide headline percentages alone

### 4.1 Workflow coverage expectations

- for workflow-heavy services, maintain a coverage matrix in the repo that maps
  important scenarios and invariants to test tiers
- keep that matrix workflow-oriented rather than file-oriented
- update it when workflow semantics or higher-tier coverage materially change

### 4.2 Smoke-test expectations

- keep a lightweight smoke script that can run from a workstation, bastion, or
  CI job without requiring the full repo build outputs
- use smoke tests to answer “does the deployed service work end to end right
  now?”, not as a substitute for deeper automated test tiers

## 5. Performance and Capacity Validation

- load test critical request paths before declaring a service production ready
  when latency, throughput, or cost matter materially
- validate queue-worker throughput and backlog recovery on workflow-heavy
  services
- exercise autoscaling assumptions with realistic traffic or job pressure
  rather than assuming default thresholds will behave well in production
- add soak or endurance tests for services that keep long-lived state,
  expensive caches, or heavy background loops
