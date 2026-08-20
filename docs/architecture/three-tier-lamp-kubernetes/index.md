# Containerized Three-Tier LAMP on Kubernetes / OpenShift

## Executive Summary

This architecture evolves the [On-Premises Three-Tier LAMP Architecture](../three-tier-lamp/index.md)
from statically provisioned VMs into a containerized platform running on
Kubernetes or OpenShift.

The application tiers stay the same — Web, Application, Database — but the
Web and Application tiers are re-platformed onto container orchestration
while the Database tier is deliberately **kept outside the cluster** for
this iteration. Running stateful databases well inside Kubernetes requires
operational maturity (storage classes, backup operators, failover testing)
that this lab has not yet built up, so DB01/DB02 continue to serve the
application exactly as they did in the VM-based lab.

This mirrors how most real migrations happen: stateless tiers move first,
stateful systems move last.

---

## Relationship to the VM-Based Architecture

```text
Three-Tier LAMP (VMs)                Three-Tier LAMP (Kubernetes)
----------------------                -----------------------------
WEB01 / WEB02  (Apache)      --->     Deployment: web  (Apache, N replicas)
APP01 / APP02  (PHP)         --->     Deployment: app  (PHP-FPM, N replicas)
DB01 / DB02    (MySQL/MariaDB)  ===   DB01 / DB02  (unchanged, external to cluster)
Load Balancer                --->     Ingress / OpenShift Route
Ansible (server config)      --->     Ansible (cluster/infra) + GitOps (app)
```

Decisions already made for the VM-based lab — three-tier separation, deny-by-default
traffic, deferred database HA technology — carry forward here. New decisions
specific to containerization are recorded in [Decisions](decisions.md).

---

## Architecture Layers

| Tier              | VM-Based Implementation | Kubernetes/OpenShift Implementation                     |
| ------------------ | ------------------------ | --------------------------------------------------------- |
| Entry Point        | Load Balancer (HAProxy/F5) | Ingress Controller (Kubernetes) or Route (OpenShift)     |
| Web Tier           | WEB01, WEB02 (Apache VMs) | `web` Deployment — Apache container, horizontally scaled |
| Application Tier   | APP01, APP02 (PHP VMs)   | `app` Deployment — PHP-FPM container, horizontally scaled |
| Database Tier      | DB01, DB02 (MySQL/MariaDB VMs) | Unchanged — accessed from the cluster via a headless/external Service |

See [Architecture](architecture.md) for the full component diagram, Kubernetes/OpenShift
object design, trust boundaries, and scaling model.

---

## Architecture Scope

This iteration focuses on:

* Containerizing the Web and Application tiers
* Cluster-native traffic entry, scaling, and self-healing
* Namespace-based network segmentation
* Carrying forward the security posture of the VM-based lab into Kubernetes-native controls

The following remain out of scope for this iteration and are noted as future
work in [Decisions](decisions.md):

* Running the Database tier inside the cluster (StatefulSet or Operator)
* Service mesh
* Multi-cluster / multi-region topology
* Progressive delivery (canary/blue-green)
