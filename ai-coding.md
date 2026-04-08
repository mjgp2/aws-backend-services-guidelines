---
title: AI Coding
permalink: /ai-coding/
---

## 1. Purpose

Use this guidance to make a service repo safe and effective for AI coding
agents.

The point is not to replace normal engineering discipline with prompts. The
point is to give agents the same things a strong new engineer would need:
clear specs, clear boundaries, clear tests, and clear repository guidance.

## 2. What Agents Need

AI coding agents work best when the repo has explicit source documents for:

- requirements and non-goals
- happy-path and failure-path flows
- state models and lifecycle rules
- API, event, and schema contracts
- test expectations and verification steps
- ownership boundaries and escalation points

If those are missing, the agent will fill the gaps with plausible guesses.
That is fine for scaffolding and low-risk refactors. It is dangerous for
behavior, architecture, and operational changes.

## 3. Minimum Documentation Surface

For meaningful service work, a good baseline is:

- a service tech design or spec document
- an operations guide or runbook
- a release or migration runbook when schema or deployment sequencing matters
- a testing guide or coverage matrix for the critical behaviors
- repository guidance for agents and contributors

Keep these documents in the service repo, close to the code they describe.

## 4. Repository Guidance Files

An `AGENTS.md` file is a good way to give repository-specific instructions to
AI coding agents.

It should usually cover:

- where the canonical design docs live
- coding and architectural constraints
- what must be verified before a task is considered complete
- which files or areas are sensitive and need extra care
- when human review is required
- what not to guess about

Treat `AGENTS.md` as an execution contract, not a substitute for the actual
service design.

## 5. Good Inputs For Agents

Before asking an agent to implement meaningful service work, make sure the repo
or task provides:

- the business or operator goal
- the relevant workflows and failure modes
- the contract or schema changes, if any
- the test shape and proof required for completion
- any migration, rollout, or rollback constraints

The better the spec surface, the more the agent can act like a reliable
implementation worker instead of a guess-heavy code generator.

## 6. What To Keep Explicit

Agents should not have to infer these from scattered code alone:

- which state transitions are valid
- which side effects must be idempotent
- which compatibility windows matter
- which tests are required for confidence
- which deployed checks prove the change actually works
- which decisions belong to product versus platform

Write those things down once in the maintained docs, then have code and tests
implement them.

## 7. Human Review Still Matters

AI coding agents can accelerate implementation, but they do not remove the need
for human review.

Human review is especially important for:

- architecture changes
- security boundaries
- data model changes
- migrations and rollout sequencing
- shared libraries and shared platform code
- IaC changes that affect more than one service

The standard should be: the agent may draft or implement, but humans still own
the technical decision and the release risk.

## 8. A Good Outcome

A good repo for AI coding is not one with the fanciest prompts. It is one
where:

- the docs make the intended behavior explicit
- the tests make the proof requirements explicit
- the repository guidance makes local rules explicit
- the ownership model makes review expectations explicit

That is also a good repo for humans.

## Related Guidance

- [Software]({{ '/software/' | relative_url }}): service tech design, flows, contracts, and test shape
- [Testing]({{ '/testing/' | relative_url }}): test strategy and coverage expectations
- [IaC]({{ '/iac/' | relative_url }}): infrastructure repo structure and review expectations
- [Team]({{ '/team-topologies/' | relative_url }}): ownership and escalation boundaries
