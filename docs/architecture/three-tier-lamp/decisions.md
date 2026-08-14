# Architecture Decisions

ADR = Architecture Decision Record.

## 1. Purpose

This document records the major architectural decisions made for the
On-Premises Three-Tier LAMP platform.

The objective is to capture:

- What decision was made
- Why the decision was made
- Alternatives considered
- Trade-offs
- Consequences

---

## ADR-001: Use Three-Tier Architecture

### Status

Accepted

### Context

The application requires separation between presentation, business logic,
and persistent data.

A single-server architecture would tightly couple these responsibilities
and make independent scaling and security controls more difficult.

### Decision

Use a three-tier architecture consisting of:

1. Web Tier
2. Application Tier
3. Database Tier

```text
User
  |
  v
Web Tier
  |
  v
Application Tier
  |
  v
Database Tier
```

### Alternatives Considered

#### Two-Tier Architecture

```text
Client
  |
  v
Application + Database
```

Rejected because the application and database responsibilities remain
tightly coupled.

#### Single-Tier Architecture

```text
Client
  |
  v
Single Server
```

Rejected because it creates a larger failure domain and limits independent
scaling.

### Consequences

#### Positive

- Clear separation of responsibilities
- Better security boundaries
- Independent scaling
- Easier troubleshooting
- Better operational control

#### Negative

- More infrastructure
- More network communication
- Higher operational complexity

---

## ADR-002: Use Linux as the Server Operating System

### Status

Accepted

### Context

The platform requires an operating system for Web, Application, and
Database servers.

The environment is intended to use enterprise Linux infrastructure.

### Decision

Use Linux for all application servers.

The initial architecture assumes an enterprise Linux distribution such as:

- Red Hat Enterprise Linux
- Rocky Linux
- AlmaLinux

The exact distribution can be selected according to organizational standards.

### Alternatives Considered

#### Microsoft Windows Server

Not selected because the application stack is based on Apache, PHP,
and MySQL/MariaDB and the target infrastructure is Linux-oriented.

### Consequences

#### Positive

- Strong support for Apache/PHP
- Mature automation ecosystem
- Strong scripting capabilities
- Excellent Ansible support
- Lower licensing cost in many deployments

#### Negative

- Requires Linux administration skills
- Enterprise support/licensing may still apply depending on distribution

---

## ADR-003: Use Apache as the Web Server

### Status

Accepted

### Context

The application requires an HTTP server to receive and process web requests.

### Decision

Use Apache HTTP Server in the Web Tier.

```text
Load Balancer
      |
      v
Apache
      |
      v
PHP Application
```

### Alternatives Considered

#### NGINX

NGINX is a valid alternative and could provide strong performance and
reverse-proxy capabilities.

It was not selected for this initial architecture because the objective
is to demonstrate a traditional LAMP implementation using Apache.

#### Microsoft IIS

Not selected because the target operating system and application stack
are Linux/PHP based.

### Consequences

#### Positive

- Mature technology
- Strong PHP integration
- Large ecosystem
- Well understood by Linux administrators

#### Negative

- May require careful tuning under high concurrency
- Requires secure configuration and regular patching

---

## ADR-004: Use PHP for the Application Tier

### Status

Accepted

### Context

The solution is intended to demonstrate a classic LAMP architecture.

### Decision

Use PHP as the application runtime.

The Application Tier consists of Linux servers running PHP.

```text
APP01
Linux + PHP

APP02
Linux + PHP
```

### Alternatives Considered

Other application runtimes such as:

- Java
- Python
- Node.js
- .NET

could be used for a modern application architecture.

They are outside the scope of this initial LAMP implementation.

### Consequences

#### Positive

- Native LAMP architecture
- Large PHP ecosystem
- Straightforward Apache integration
- Mature application framework options

#### Negative

- Requires PHP-specific application security practices
- Application dependencies must be continuously maintained

---

## ADR-005: Use MySQL/MariaDB as the Database

### Status

Accepted

### Context

The application requires a relational database for persistent data.

### Decision

Use MySQL or MariaDB.

The initial logical architecture supports either implementation.

```text
APP01 ──┐
        ├──> DB01
APP02 ──┘
```

### Alternatives Considered

#### PostgreSQL

PostgreSQL is a strong enterprise relational database alternative.

It was not selected because the objective of this project is to demonstrate
a traditional LAMP stack.

#### NoSQL Database

NoSQL databases are not required for the initial relational application
model.

### Consequences

#### Positive

- Mature relational database
- Strong PHP support
- Well understood operational model
- Broad ecosystem

#### Negative

- Database HA requires careful architecture
- Stateful systems are more difficult to scale than stateless application tiers

---

## ADR-006: Use a Load Balancer

### Status

Accepted

### Context

Multiple Web and Application servers are required for availability and
horizontal scalability.

Directly exposing individual Web servers would make traffic management
and failure handling more difficult.

### Decision

Introduce a Load Balancer in front of the Web Tier.

```text
                Users
                  |
                  v
            Load Balancer
              /       \
             /         \
          WEB01       WEB02
```

### Alternatives Considered

#### DNS Round Robin

Rejected as the primary load-balancing mechanism because DNS does not
provide reliable real-time health-based traffic management.

#### Direct Client Access

Rejected because it exposes individual servers and creates a larger
availability risk.

### Consequences

#### Positive

- Health checks
- Traffic distribution
- Server failover
- Horizontal scalability
- Centralized entry point

#### Negative

- Load Balancer becomes a potential single point of failure
- HA must be designed for production deployments

---

## ADR-007: Separate Network Zones by Tier

### Status

Accepted

### Context

The Web, Application, and Database tiers have different security
requirements.

The Database Tier contains sensitive persistent information and should
not be directly accessible by users.

### Decision

Use separate network zones.

```text
DMZ
 |
 v
Web Zone
 |
 v
Application Zone
 |
 v
Database Zone
```

### Alternatives Considered

#### Single Network

Rejected because it provides insufficient isolation between application
components.

### Consequences

#### Positive

- Reduced attack surface
- Controlled east-west traffic
- Better firewall policy
- Easier security monitoring

#### Negative

- More network configuration
- Requires firewall rule management

---

## ADR-008: Deny Network Traffic by Default

### Status

Accepted

### Context

Allowing unrestricted communication between application tiers would
increase the potential impact of a compromised server.

### Decision

Use a deny-by-default firewall policy.

Only required communication paths are allowed.

Example:

```text
User
 |
 | 443
 v
Load Balancer
 |
 | 80/443
 v
Web Tier
 |
 | Application Port
 v
Application Tier
 |
 | 3306
 v
Database Tier
```

### Consequences

#### Positive

- Reduced attack surface
- Stronger segmentation
- Easier security auditing

#### Negative

- Requires careful dependency mapping
- Incorrect firewall rules can affect application availability

---

## ADR-009: Use Horizontal Scaling for Web and Application Tiers

### Status

Accepted

### Context

Web and application workloads may increase over time.

Vertical scaling alone would eventually reach physical or operational
limitations.

### Decision

Scale Web and Application tiers horizontally.

```text
WEB01
WEB02
WEB03
...
WEB-N
```

and:

```text
APP01
APP02
APP03
...
APP-N
```

### Consequences

#### Positive

- Improved scalability
- Improved availability
- Easier capacity expansion

#### Negative

- Requires load balancing
- Application state must be handled appropriately
- Configuration must be consistent across servers

---

## ADR-010: Prefer Stateless Application Design

### Status

Accepted

### Context

Multiple Application Tier servers require requests to be processed by
any available application server.

Local session storage can create dependency on a specific server.

### Decision

Design the PHP application to be as stateless as practical.

Where sessions are required, use shared or distributed session storage.

Possible options include:

- Database-backed sessions
- Distributed cache
- Shared session store

### Consequences

#### Positive

- Easier horizontal scaling
- Better failover
- Reduced dependency on individual servers

#### Negative

- Requires additional session-management architecture

---

## ADR-011: Use Ansible for Configuration Management

### Status

Accepted

### Context

Multiple Linux servers need consistent configuration.

Manual configuration would create configuration drift.

### Decision

Use Ansible for Linux server configuration and application deployment
automation.

Example:

```text
Ansible Control Node
        |
        +---- WEB01
        +---- WEB02
        +---- APP01
        +---- APP02
        +---- DB01
        +---- DB02
```

### Consequences

#### Positive

- Repeatable configuration
- Reduced manual work
- Configuration consistency
- Idempotent automation
- Version-controlled automation

#### Negative

- Requires Ansible expertise
- Playbooks must be maintained

---

## ADR-012: Version Control Architecture Documentation

### Status

Accepted

### Context

Architecture documentation and diagrams can become outdated if maintained
separately from implementation.

### Decision

Store architecture documentation, diagrams, and related source files in Git.

The repository is:

```text
architecture-lab
```

MkDocs is used to publish the documentation.

### Consequences

#### Positive

- Version history
- Peer review
- Traceability
- Easy collaboration
- Documentation can evolve with the architecture

#### Negative

- Documentation requires maintenance
- Poorly maintained documentation can still become outdated

---

## ADR-013: Use MkDocs for Architecture Documentation

### Status

Accepted

### Context

The architecture requires a centralized documentation portal that can be
version controlled and automatically published.

### Decision

Use MkDocs with the Material theme.

```text
Markdown
    |
    v
MkDocs
    |
    v
Static Website
    |
    v
GitHub Pages
```

### Alternatives Considered

- Confluence
- GitHub Wiki
- Static HTML
- Other documentation generators

MkDocs was selected because architecture documentation remains close to
the source repository and can be automatically deployed.

### Consequences

#### Positive

- Markdown-based
- Git-friendly
- Easy local development
- Supports Mermaid diagrams
- Can be automatically deployed

#### Negative

- Requires Markdown knowledge
- Requires CI/CD configuration

---

## ADR-014: Use Mermaid for Architecture Diagrams

### Status

Accepted

### Context

Architecture diagrams should be version controlled and maintainable.

Binary diagram files can make change tracking more difficult.

### Decision

Use Mermaid as the primary diagram-as-code format for diagrams that can
be represented effectively using Mermaid.

### Consequences

#### Positive

- Text-based
- Version controlled
- Easy to modify
- Automatically rendered by documentation tooling

#### Negative

- Complex visual diagrams may be easier to create with Draw.io
- Mermaid has limitations for highly detailed infrastructure diagrams

Draw.io or other diagramming tools may be introduced when Mermaid is not
appropriate.

---

## ADR-015: Defer Database HA Technology Selection

### Status

Accepted

### Context

The architecture currently contains DB01 and DB02, but selecting the
exact database HA mechanism requires additional business and technical
requirements.

### Decision

Do not prematurely select a specific MySQL/MariaDB HA implementation.

The following options remain under consideration:

- Primary/Replica
- Primary/Standby
- Synchronous replication
- MariaDB Galera
- MySQL Group Replication
- Other enterprise-supported solutions

### Decision Criteria

The final decision will consider:

- RPO
- RTO
- Transaction consistency
- Performance
- Operational complexity
- Existing enterprise skills
- Licensing
- Monitoring
- Backup and recovery
- Failover requirements

### Consequence

The current architecture remains technology-neutral regarding the exact
database HA mechanism until the requirements are defined.

---

## 16. Architecture Decision Summary

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | Three-tier architecture | Accepted |
| ADR-002 | Linux operating system | Accepted |
| ADR-003 | Apache Web Server | Accepted |
| ADR-004 | PHP Application Runtime | Accepted |
| ADR-005 | MySQL/MariaDB | Accepted |
| ADR-006 | Load Balancer | Accepted |
| ADR-007 | Network segmentation | Accepted |
| ADR-008 | Deny-by-default firewall | Accepted |
| ADR-009 | Horizontal scaling | Accepted |
| ADR-010 | Stateless application design | Accepted |
| ADR-011 | Ansible automation | Accepted |
| ADR-012 | Git-based documentation | Accepted |
| ADR-013 | MkDocs | Accepted |
| ADR-014 | Mermaid diagrams | Accepted |
| ADR-015 | Database HA selection deferred | Accepted |

---

## 17. Future Architecture Decisions

Additional ADRs should be created when decisions are made regarding:

- Load Balancer implementation
- Firewall platform
- Linux distribution
- PHP version
- PHP-FPM architecture
- MySQL vs MariaDB
- Database replication technology
- Storage architecture
- Backup platform
- Monitoring platform
- Centralized logging platform
- Identity and access management
- Secrets management
- TLS certificate management
- Disaster Recovery architecture
- CI/CD platform