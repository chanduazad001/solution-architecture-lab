# On-Premises Three-Tier LAMP Architecture

## Executive Summary

This architecture demonstrates a production-oriented three-tier web application platform running on on-premises Linux infrastructure.

The solution separates the application into three logical tiers:

* **Web Tier** — Linux + Apache
* **Application Tier** — Linux + PHP
* **Database Tier** — Linux + MySQL/MariaDB

The architecture is designed to provide scalability, availability, security, and operational maintainability.

---

## Architecture Layers

The solution separates presentation, business logic, and persistent data
into three tiers, front-ended by a load balancer:

| Tier              | Technology              | Servers       |
| ----------------- | ------------------------ | ------------- |
| Web Tier          | Linux + Apache           | WEB01, WEB02  |
| Application Tier  | Linux + PHP              | APP01, APP02  |
| Database Tier     | Linux + MySQL/MariaDB    | DB01, DB02    |

The Application and Database tiers are isolated from direct user access.

See [Architecture](architecture.md) for the full component diagram, tier
responsibilities, communication paths, and trust boundaries.

---

## Request Flow

A typical application request follows this logical path:

```text
User
  ↓
Load Balancer
  ↓
Web Tier
  ↓
Application Tier
  ↓
Database Tier
  ↓
Application Tier
  ↓
Web Tier
  ↓
Load Balancer
  ↓
User
```

---

## Key Architecture Principles

### Separation of Concerns

Each tier has a clearly defined responsibility.

### Horizontal Scalability

Additional Web or Application servers can be added when application demand increases.

### Network Segmentation

Each tier should be placed in an appropriate network zone with controlled communication between tiers.

### Security

Users should never communicate directly with the Application or Database tiers.

### High Availability

Multiple Web and Application servers reduce the impact of individual server failures.

### Observability

The platform should provide infrastructure monitoring, application monitoring, centralized logging, and alerting.

### Automation

Server configuration and application deployment should be automated using tools such as Ansible and CI/CD pipelines.

---

## Technology Stack

| Layer                    | Technology         |
| ------------------------ | ------------------ |
| Client                   | Web Browser        |
| Load Balancer            | HAProxy / F5       |
| Operating System         | Linux              |
| Web Server               | Apache HTTP Server |
| Application              | PHP                |
| Database                 | MySQL / MariaDB    |
| Configuration Management | Ansible            |
| Source Control           | Git / GitHub       |
| Documentation            | MkDocs             |

---

## Architecture Scope

This initial architecture focuses on the core three-tier application platform.

The following concerns will be designed separately:

* Network architecture
* Security controls
* Firewall rules
* Load-balancing strategy
* Database high availability
* Monitoring and logging
* Backup and recovery
* Disaster recovery
* Deployment automation
* Infrastructure automation

These concerns are documented in the subsequent architecture sections.