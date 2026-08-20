# Architecture

Each architecture in this lab is a deliberate case study, documented with
the same discipline: **Overview → Architecture → Data Flow → Security →
Availability → Decisions**. Later labs build on earlier ones rather than
starting from a blank page — an ADR in one lab often extends or supersedes
a decision made in another.

---

## Labs

| Architecture | Description | Status |
| --------------- | -------------- | -------- |
| [Three-Tier LAMP](three-tier-lamp/index.md) | On-premises three-tier web application (Apache, PHP, MySQL/MariaDB) deployed across Linux VMs, with a load balancer, network segmentation, and 15 ADRs covering the foundational decisions. | Active |
| [Three-Tier LAMP on Kubernetes](three-tier-lamp-kubernetes/index.md) | Evolves the VM-based LAMP app by containerizing the Web and Application tiers onto Kubernetes/OpenShift, while the Database tier stays on the existing VMs. | Active |

---

## How the Labs Relate

```text
Three-Tier LAMP (VMs)
        │
        │ containerize Web + Application tiers
        ▼
Three-Tier LAMP on Kubernetes
        │
        │ planned
        ▼
   (see Coming Next)
```

---

## Coming Next

Planned additions, in rough priority order — not yet documented:

* **DevSecOps CI/CD pipeline** for the containerized LAMP app — build, scan, sign, deploy
* **Observability stack** (metrics, logs, tracing) layered onto the Kubernetes lab
* **RAG / GenAI architecture** — vector database, embedding pipeline, LLM gateway
* **Cloud landing zone** (Azure or AWS) — networking, IAM, and guardrails foundation

See each lab's own **Decisions** page for the specific future ADRs it anticipates.
