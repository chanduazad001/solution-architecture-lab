# Architecture Decision Making

## 1. Why Record Decisions?

Every earlier topic in this track ends in a trade-off, not a single right
answer — SQL or NoSQL, sync or async, one big service or many small ones.
An **Architecture Decision Record (ADR)** captures which way a specific
decision went, in a specific context, and why — so the reasoning survives
past the moment it was made, and a later reader (including a future
version of the person who wrote it) can tell whether the context that
justified it still holds.

This page follows the exact ADR format used throughout the
[Architecture](../../architecture/index.md) track — see, for example,
[Three-Tier LAMP: ADR-001](../../architecture/three-tier-lamp/decisions.md#adr-001-use-three-tier-architecture).
The difference is scope: those ADRs record decisions for one specific,
real platform; the ones below are worked examples of the *reasoning*
behind common system design trade-offs, meant to be adapted to a real
system's actual context rather than applied as-is.

---

## ADR-D001: SQL vs NoSQL for the Primary Datastore

### Status

Illustrative (worked example, not tied to a specific system)

### Context

A new service needs a primary datastore. It has multi-record transactional
requirements (an order and its line items must be created atomically), a
moderate write volume well within a single well-sized instance's capacity,
and ad hoc reporting queries are expected from day one.

### Decision

Use a relational (SQL) database as the primary datastore.

### Alternatives Considered

#### Document Database

```text
Application
    |
    v
Document Store
```

Rejected: the transactional requirement (atomic multi-record writes)
would require either restructuring the data into a single document per
transaction — awkward given the actual access patterns — or giving up the
atomicity guarantee entirely.

#### Key-Value Store

```text
Application
    |
    v
Key-Value Store
```

Rejected: no support for the ad hoc reporting queries required; would
need a second system for that purpose from day one, adding operational
complexity the requirements don't yet justify.

### Consequences

#### Positive

- ACID transactions satisfy the atomic multi-record requirement directly
- Ad hoc queries and joins are supported natively
- Well-understood operational model

#### Negative

- Horizontal write scaling requires sharding later if write volume
  outgrows a single instance (see [Scalability §6](../scalability/index.md#6-scaling-the-database-partitioning-sharding))
- Schema changes require more care than a schemaless document store

---

## ADR-D002: Managed Redis for the Caching Layer

### Status

Illustrative (worked example)

### Context

Read load on the primary database (from ADR-D001) is now the dominant
cost driver, and the estimation numbers (see
[Requirements & Estimation](../requirements-estimation/index.md)) show a
read/write ratio heavily skewed toward reads. A caching layer is needed
that's shared across all application instances.

### Decision

Use a managed, distributed Redis cluster as a cache-aside layer (see
[Caching §3](../caching/index.md#3-cache-strategies)) in front of the
primary database.

### Alternatives Considered

#### In-Process (Local) Cache

```text
App Instance
    |
    v
In-Memory Cache (local to this instance only)
```

Rejected: not shared across instances — each instance would need its own
copy, multiplying database load on a cold start and creating instance-
specific staleness after invalidation.

#### Self-Managed Cache Cluster

```text
App
  |
  v
Self-Hosted Cache Cluster
```

Rejected for this context: the operational burden of running and
patching a cache cluster wasn't justified given the team's current
capacity — revisit if the managed offering's cost or feature limits
become the binding constraint.

### Consequences

#### Positive

- Shared across all application instances — no per-instance cold-start
  penalty
- Managed service removes patching/failover operational burden
- Directly reduces database read load, per
  [Scalability §8](../scalability/index.md#8-caching-and-cdn-for-scale)

#### Negative

- Introduces a second source of truth with its own staleness/invalidation
  risk (see [Caching §8](../caching/index.md#8-common-mistakes))
- Adds a network dependency the application didn't previously have

---

## ADR-D003: Kafka-Style Event Streaming vs RabbitMQ-Style Queue

### Status

Illustrative (worked example)

### Context

A new "order placed" event needs to reach three independent downstream
consumers (fulfillment, analytics, notifications), and analytics needs to
be able to reprocess historical events when its aggregation logic changes.

### Decision

Use an event streaming platform (Kafka-style, per
[Messaging & Events §4](../messaging-events/index.md#4-producers-consumers-topics-partitions-consumer-groups))
rather than a traditional message queue.

### Alternatives Considered

#### Traditional Message Queue (RabbitMQ-style)

```text
Producer
    |
    v
Queue
    |
    v
One Consumer (per message)
```

Rejected: a queue delivers each message to one consumer, not three — fan-
out would require publishing to three separate queues, and none would
support the analytics team's replay requirement, since consumed messages
are deleted.

### Consequences

#### Positive

- Native fan-out to multiple independent consumer groups, per
  [Messaging & Events §3](../messaging-events/index.md#3-message-queues-vs-pubsub-vs-event-streaming)
- Retained log supports analytics replaying history after logic changes

#### Negative

- Higher operational complexity than a simple queue (partition management,
  consumer group rebalancing)
- Ordering is only guaranteed within a partition, requiring a deliberate
  partition key (see [Messaging & Events §5](../messaging-events/index.md#5-ordering-guarantees))

---

## ADR-D004: Synchronous vs Asynchronous Order Confirmation

### Status

Illustrative (worked example)

### Context

After an order is placed, a confirmation email must be sent. The checkout
API's response time is a tracked non-functional requirement (p99 < 300ms);
email delivery, by contrast, is not something the user is waiting on
screen for.

### Decision

Send the confirmation email asynchronously, via a message queue, rather
than as part of the synchronous checkout request.

### Alternatives Considered

#### Synchronous Call to the Email Service

```text
Checkout Request
    |
    v
Charge Payment
    |
    v
Send Email (blocking)
    |
    v
Response to User
```

Rejected: ties checkout latency and availability to the email service's —
a slow or down email provider would make checkout itself slow or fail,
for functionality the user isn't blocked on.

### Consequences

#### Positive

- Checkout latency is decoupled from email delivery latency
- Email service being temporarily down doesn't fail checkout

#### Negative

- Email delivery is now eventually consistent — a user could theoretically
  complete checkout slightly before the confirmation arrives
- Requires the queue-based failure handling (retry, DLQ) covered in
  [Messaging & Events §7](../messaging-events/index.md#7-dead-letter-queues-retry-backoff)

---

## ADR-D005: Modular Monolith vs Microservices for a New Product

### Status

Illustrative (worked example)

### Context

A new product is being built by a single team of six engineers. There is
no current requirement for independent deploy cadences or independent
scaling between its components.

### Decision

Build a modular monolith with clear internal module boundaries, not
microservices.

### Alternatives Considered

#### Microservices from Day One

```text
Service A   Service B   Service C
    |           |           |
    v           v           v
  DB A        DB B        DB C
```

Rejected: with one team and no current independent-scaling or independent-
deploy requirement, this trades a real, current cost (distributed
tracing, network calls, multiple deployables, per
[Design Patterns §4](../design-patterns/index.md#4-microservices-vs-modular-monolith))
for a benefit the team doesn't need yet.

### Consequences

#### Positive

- Single deployable — simpler CI/CD, simpler local development
- In-process calls between modules — no network latency or partial-failure
  handling required internally
- Module boundaries still enforced in code, keeping a future extraction
  to microservices possible if the team or scale later requires it

#### Negative

- All modules currently share a release cadence
- A bug in one module can still affect the whole process, unlike a fully
  isolated service

---

## ADR-D006: REST vs gRPC for Internal Service Calls

### Status

Illustrative (worked example)

### Context

Two internal services, both written by teams within the same
organization, need to communicate frequently and with low latency; there
are no external or third-party callers of this particular interface.

### Decision

Use gRPC for this internal service-to-service call.

### Alternatives Considered

#### REST over JSON

```text
Service A
    |
    v (JSON over HTTP)
Service B
```

Rejected for this specific internal call: JSON serialization overhead and
REST's lack of native streaming were measurable costs for this
latency-sensitive, high-frequency path — but REST remains the right choice
for this system's external-facing API, per
[APIs & Communication §3](../apis-communication/index.md#3-rest-grpc-graphql),
since broad client compatibility matters there and didn't here.

### Consequences

#### Positive

- Lower serialization overhead and native streaming support
- Strongly-typed contract (Protocol Buffers) catches interface mismatches
  at build time

#### Negative

- Less human-readable on the wire, harder to debug with generic HTTP
  tooling than JSON
- Requires both services to share/generate the same `.proto` definitions,
  adding a build-time dependency between them

---

## 2. Related Concepts

* [Architecture](../../architecture/index.md) — the ADR format above
  applied to specific, real platforms rather than worked examples
* [Case Studies](../case-studies/index.md) — the
  [URL Shortener's own decisions](../case-studies/url-shortener/decisions.md)
  page shows this same format applied to one specific system's real
  constraints, rather than the illustrative examples above
* Every topic referenced in the ADRs above — each one covers its side of
  the trade-off in more depth than an ADR's Context/Consequences sections
  are meant to
