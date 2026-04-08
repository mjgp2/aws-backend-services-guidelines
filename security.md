---
title: Security
permalink: /security/
---

## 1. Purpose

Use this guidance for backend security posture across application, runtime,
infrastructure, and deployment boundaries.

[Auth]({{ '/auth/' | relative_url }}) covers bearer-token and caller-identity
posture. This page covers the broader baseline around credentials, image
scanning, account guardrails, data handling, and review expectations.

## 2. Default Security Posture

- least privilege is the default for IAM, network reachability, secret access,
  and operator permissions
- default new services to zero application-managed AWS access keys
- prefer IAM roles, IAM DB auth, and short-lived federated deploy access
- keep security controls in code and reviewed change paths, not in remembered
  console state
- treat security as a normal design concern, not a late compliance add-on
- make the secure path the easy path through shared platform defaults

## 3. Credentials, Secrets, And Identity

- use SSM Parameter Store for non-secret runtime metadata
- use Secrets Manager only for secrets that cannot be designed away
- keep long-lived admin credentials limited to bootstrap or emergency use
- use OIDC or another short-lived federated mechanism for CI deploy access
- record every remaining secret with an owner, rotation plan, and failure mode
- treat public verification keys such as JWKS material as metadata, not secrets

## 4. Secret Scanning And Dependency Scanning

- pre-commit filters should include secret scanning so obvious credential leaks
  are blocked before commit
- CI should also run secret scanning and fail closed on real findings
- dependency scanning should run in CI for application and infrastructure repos
- teams should review dependency findings with the same seriousness as other
  externally reported vulnerabilities, not leave them as background noise

Local hooks are an early filter, not the only security gate.

## 5. Container And Artifact Scanning

If services ship container images, image scanning should be part of the normal
delivery posture.

- enable ECR image scanning for service images
- treat high-severity or critical image findings as release blockers unless the
  team has a documented reason and mitigation
- keep base images current enough that image scanning findings can actually be
  remediated in routine engineering work
- do not assume an image is safe just because the application code diff was
  small; base-image and OS-package drift still matters

Image scanning is not a substitute for dependency scanning, and dependency
scanning is not a substitute for image scanning. Both catch different classes
of risk.

## 6. Account And Platform Guardrails

For AWS account posture, a good default platform baseline includes:

- Security Hub enabled in the account and Region
- AWS Config or an equivalent configuration recording posture
- IAM Access Analyzer or equivalent external-access analysis
- central review of meaningful findings rather than silent accumulation

These services are useful only if findings lead to action. The goal is not to
maximize dashboard count. The goal is to detect dangerous drift and exposure
early enough to fix it.

## 7. Runtime And Network Guardrails

- keep long-lived runtimes in private subnets by default
- review security-group reachability through IaC changes, not ad hoc console
  edits
- give each runtime only the IAM permissions and network ingress it actually
  needs
- start from deny-by-default and add only the access needed for the concrete
  runtime, workflow, or operator path
- separate operator access from public traffic paths
- assume internal network location alone is not proof of authorization

## 8. Data And Logging Guardrails

- classify data well enough to distinguish public, internal, confidential, and
  regulated data
- define whether sensitive data may appear in logs, traces, events, caches,
  and object metadata before launch
- make retention and deletion policy explicit for security-sensitive and
  compliance-sensitive data
- keep search, analytics, and derived data stores aligned to the same data
  handling rules as the source systems feeding them

## 9. Review Expectations

- platform should review changes to shared IAM, networking, guardrails,
  account-level security services, and shared IaC
- service teams still own the application-level threat model, authorization
  logic, and safe use of the platform controls
- destructive security exceptions should be explicit, reviewed, and documented

Security review should protect the platform and the product without creating a
standing central queue for routine safe changes.

## Related Guidance

- [Auth]({{ '/auth/' | relative_url }}): JWT, JWKS, and caller-identity posture
- [Infra]({{ '/infra/' | relative_url }}): deployment, IAM, secrets, and network defaults
- [IaC]({{ '/iac/' | relative_url }}): IaC review posture and guardrail tooling
- [Testing]({{ '/testing/' | relative_url }}): CI, smoke, and pre-commit verification layers
- [Checklist]({{ '/new-service-checklist/' | relative_url }}): minimum launch-readiness bar
