# Architecture Decisions

ADR = Architecture Decision Record.

## 1. Purpose

This document records the architectural decisions specific to
containerizing the three-tier LAMP application on Kubernetes/OpenShift.

These ADRs extend, and in some cases supersede, the decisions recorded in
the [VM-based lab's decisions](../three-tier-lamp/decisions.md). Where a
decision changes something previously accepted there, it is called out
explicitly.

Numbering restarts at ADR-001 for this document — each architecture keeps
its own decision log, cross-referenced by name where relevant.

---

## ADR-001: Containerize the Web and Application Tiers on Kubernetes/OpenShift

### Status

Accepted

### Context

The VM-based Web and Application tiers (three-tier-lamp ADR-002, ADR-003,
ADR-004) are stateless and horizontally scaled, which makes them good
candidates for container orchestration. Manual VM provisioning for scaling
events is slower and more operationally expensive than platform-native
scaling.

### Decision

Run the Web tier (Apache) and Application tier (PHP-FPM) as Kubernetes
Deployments, orchestrated by Kubernetes or OpenShift.

### Alternatives Considered

#### Continue with VM-based scaling only

Rejected as the long-term direction because it requires manual server
provisioning for every scaling event and does not provide self-healing.

#### Serverless / FaaS for the Application tier

Not selected for this iteration; would require significant application
restructuring and is a larger architectural change than a lift-and-shift
containerization.

### Consequences

#### Positive

- Native self-healing and horizontal scaling
- Faster, more consistent deployments
- Declarative, version-controlled workload definitions

#### Negative

- Requires Kubernetes/OpenShift operational expertise
- Introduces a new failure surface (cluster control plane, networking)

---

## ADR-002: Keep the Database Tier Outside the Cluster (Initially)

### Status

Accepted

### Context

Running stateful databases inside Kubernetes requires storage classes,
backup/restore tooling, and failover automation that this lab has not yet
established. Three-tier-lamp ADR-015 already deferred selecting a specific
database HA technology.

### Decision

DB01/DB02 remain on their existing VMs, exposed to the cluster through a
`Service` (no selector + managed `Endpoints`, or `ExternalName`).

### Alternatives Considered

#### StatefulSet with Persistent Volumes

Rejected for this iteration — viable in principle, but requires storage
and backup automation that is not yet in place.

#### Database Operator (e.g., MySQL Operator, Percona XtraDB Cluster Operator)

Rejected for this iteration for the same reason; tracked as a future
enhancement once cluster storage and operational practices mature.

### Consequences

#### Positive

- No change to the database's operational model or existing HA decisions
- Reduces the scope and risk of this migration step
- Keeps three-tier-lamp ADR-015 valid without modification

#### Negative

- The Application tier depends on connectivity out of the cluster to the
  existing VM network, which must remain reachable and correctly firewalled
- Database availability is not managed by the same platform as the
  application, so the two must be operated with separate tooling

---

## ADR-003: Preserve Web/Application Tier Separation as Independent Deployments

### Status

Accepted

### Context

A common containerization pattern combines a web server and application
runtime (e.g., Apache + PHP-FPM) into a single Pod as two containers,
communicating over localhost.

### Decision

Keep the Web tier and Application tier as **separate Deployments and
Services**, matching the tier separation established in three-tier-lamp
ADR-001.

### Alternatives Considered

#### Single Pod with Apache and PHP-FPM as sidecar containers

Rejected because it would couple the two tiers' scaling and would blur
the security boundary between them (three-tier-lamp ADR-001's stated
rationale for separation — independent scaling and clearer security
boundaries — still applies in Kubernetes).

### Consequences

#### Positive

- Web and Application tiers scale independently via separate HPAs
- `NetworkPolicy` can enforce a clear boundary between the two tiers
- Consistent with the reasoning already accepted in three-tier-lamp ADR-001

#### Negative

- Slightly more Kubernetes objects to manage than a combined-Pod approach
- One additional network hop (Service-to-Service) versus localhost

---

## ADR-004: Use Kubernetes NetworkPolicies for East-West Traffic Control

### Status

Accepted

### Decision

Enforce a namespace-wide default-deny `NetworkPolicy`, with explicit allow
rules for Ingress→web, web→app, and app→DB01/DB02 traffic only.

### Context

This directly realizes the deny-by-default posture already accepted in
three-tier-lamp ADR-008, expressed as Kubernetes-native objects instead of
perimeter firewall rules.

### Consequences

#### Positive

- Same security posture as the VM-based lab, enforced closer to the workload
- Policy changes are version-controlled alongside the application manifests

#### Negative

- Requires the cluster's CNI plugin to support `NetworkPolicy` enforcement
- Misconfigured policies can silently break inter-tier communication

---

## ADR-005: Use Ingress/OpenShift Route as the Cluster Entry Point

### Status

Accepted

### Context

Three-tier-lamp ADR-006 introduced a Load Balancer as the entry point for
the VM-based Web tier.

### Decision

Use an Ingress Controller (Kubernetes) or Route (OpenShift) as the
in-cluster entry point. An upstream enterprise load balancer or DNS layer
may still front the cluster nodes at L4; that remains an infrastructure
concern outside this application architecture.

### Consequences

#### Positive

- Cluster-native health-based routing to `web-svc`
- TLS termination and routing rules are version-controlled with the app

#### Negative

- Adds a Kubernetes/OpenShift-specific concept that VM-based operators
  must learn
- Ingress/Route behavior differs slightly between Kubernetes distributions

---

## ADR-006: Use Kubernetes Secrets for Credentials, Defer External Secrets Integration

### Status

Accepted

### Context

Three-tier-lamp's security architecture already flagged that secrets
should never be stored in Git and recommended a centralized secrets
platform (Vault or equivalent) as a future step.

### Decision

Use native Kubernetes `Secret` objects for database credentials in this
iteration, with encryption at rest enabled at the cluster level. Defer
integrating an external secrets manager.

### Alternatives Considered

#### External Secrets Operator / Vault integration from day one

Not selected for this iteration to keep the initial migration scope
smaller; the interim approach (native Secrets + RBAC restriction) is an
accepted stepping stone, not a permanent security stance.

### Consequences

#### Positive

- Simple to implement with what the cluster already provides
- No new external dependency required to complete this migration

#### Negative

- Native `Secret` objects are base64-encoded, not encrypted, without
  cluster-level encryption-at-rest configuration
- Will require a follow-up migration once an external secrets manager is adopted

---

## ADR-007: Use HorizontalPodAutoscaler for Web/Application Scaling

### Status

Accepted

### Context

Three-tier-lamp ADR-009 established horizontal scaling as the strategy for
Web and Application tiers, implemented manually by provisioning additional
VMs.

### Decision

Use `HorizontalPodAutoscaler` for both the `web` and `app` Deployments,
scaling independently based on CPU utilization initially.

### Consequences

#### Positive

- Automatic scaling response to load, without manual provisioning
- Directly implements the strategy already accepted in three-tier-lamp ADR-009

#### Negative

- Requires correctly sized resource `requests` to make CPU-based scaling
  meaningful
- Custom-metric scaling (e.g., requests-per-second) requires additional
  metrics infrastructure not yet in place

---

## ADR-008: Adopt GitOps (Argo CD) for Application Deployment, Retain Ansible for Infrastructure

### Status

Accepted

### Context

Three-tier-lamp ADR-011 established Ansible for configuration management
and deployment automation. Ansible remains well suited to provisioning
cluster nodes and infrastructure, but Kubernetes application deployments
are better served by a declarative, reconciliation-based approach.

### Decision

Use Ansible for cluster/node-level infrastructure provisioning, and adopt
a GitOps tool (Argo CD) to continuously reconcile the `lamp-app` namespace
against manifests stored in Git.

### Alternatives Considered

#### Ansible for application deployment too

Rejected as the primary application-deployment mechanism going forward —
push-based deployment does not self-correct configuration drift the way a
GitOps reconciliation loop does.

### Consequences

#### Positive

- Git becomes the single source of truth for what is actually running
- Automatic drift detection and correction
- Clear audit trail of deployment changes via Git history

#### Negative

- Introduces a new tool and operational responsibility
- Requires the team to adopt GitOps workflows and repository conventions

---

## ADR-009: Target Both Kubernetes-Native and OpenShift-Native Object Types

### Status

Accepted

### Context

The lab's stated focus areas include both Kubernetes and OpenShift. Some
objects (`Ingress` vs. `Route`, Pod Security Admission vs. SCC) differ
between the two.

### Decision

Design the architecture around Kubernetes-native objects (`Deployment`,
`Service`, `NetworkPolicy`, `HPA`) that work unmodified on both platforms,
and explicitly document the platform-specific equivalent where the two
diverge (see [Architecture, Section 7](architecture.md#7-cluster-ingress)).

### Consequences

#### Positive

- One architecture document serves both target platforms
- Avoids OpenShift-only or vanilla-Kubernetes-only lock-in where avoidable

#### Negative

- Some OpenShift-specific capabilities (S2I builds, SCCs) are documented
  as options rather than adopted outright, requiring a follow-up decision
  once a target platform is finalized

---

## 2. Architecture Decision Summary

| ADR    | Decision                                                        | Status   |
| ------ | ------------------------------------------------------------------ | -------- |
| ADR-001 | Containerize Web and Application tiers on Kubernetes/OpenShift    | Accepted |
| ADR-002 | Keep Database tier outside the cluster (initially)                | Accepted |
| ADR-003 | Preserve Web/Application tier separation as independent Deployments | Accepted |
| ADR-004 | Use NetworkPolicies for east-west traffic control                  | Accepted |
| ADR-005 | Use Ingress/Route as the cluster entry point                       | Accepted |
| ADR-006 | Use Kubernetes Secrets, defer external secrets integration          | Accepted |
| ADR-007 | Use HorizontalPodAutoscaler for scaling                             | Accepted |
| ADR-008 | Adopt GitOps for application deployment, retain Ansible for infra   | Accepted |
| ADR-009 | Target both Kubernetes-native and OpenShift-native object types     | Accepted |

---

## 3. Future Architecture Decisions

Additional ADRs should be created when decisions are made regarding:

- Bringing the Database tier under cluster-managed HA
- Target platform finalization (Kubernetes distribution vs. OpenShift)
- External secrets manager selection
- Service mesh adoption
- Admission control / policy engine (OPA/Gatekeeper vs. Kyverno)
- Observability stack (metrics, logging, tracing)
- Multi-cluster or multi-region topology
