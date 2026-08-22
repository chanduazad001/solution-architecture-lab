# Data & Databases

This page covers database fundamentals and how to choose between them.
The mechanics of *distributing* data once you've chosen — partitioning,
sharding, replication, consistency — are covered in
[Scalability](../scalability/index.md) and
[Distributed Systems](../distributed-systems/index.md); this page focuses
on what's specific to data modeling and database selection.

## 1. What is Database Architecture?

Database architecture is the set of decisions about how data is modeled,
stored, and accessed: which database family fits the access pattern, how
strongly consistency and durability are enforced, and how the schema
supports the queries the system actually needs to run.

## 2. Why Database Choice Matters So Much

Of all the [Core Building Blocks](../building-blocks/index.md), the
database is the hardest to change after the fact — it's usually the
system's source of truth, and migrating a live data model without
downtime or data loss is one of the most expensive things a team can be
asked to do. Getting this decision closer to right the first time pays for
itself repeatedly.

## 3. Relational Databases

Relational (SQL) databases store data in tables with defined schemas and
relationships, and are built around **ACID** transactions:

| Property | Guarantee |
| -------- | ----------- |
| Atomicity | A transaction fully completes or fully rolls back — no partial writes |
| Consistency | A transaction moves the database from one valid state to another, respecting constraints |
| Isolation | Concurrent transactions don't see each other's uncommitted changes |
| Durability | Once committed, a transaction survives a crash |

**Indexes** speed up lookups on specific columns at the cost of slower
writes (every index must be updated) and extra storage. **Joins** combine
normalized tables at query time — powerful, but expensive at scale, which
is why highly-scaled relational systems often selectively denormalize.

| | Normalization | Denormalization |
| --- | --- | --- |
| Data duplication | Minimized | Deliberate, for read speed |
| Write complexity | Lower — update one place | Higher — update every copy |
| Read complexity | Higher — requires joins | Lower — data is already shaped for the query |
| Best for | Write-heavy, consistency-critical data | Read-heavy access patterns |

## 4. BASE — the NoSQL Counterpart to ACID

Where relational databases are built around ACID (§3), most NoSQL
databases are built around **BASE**, trading strong consistency for
availability and scale — a direct application of the CAP trade-off from
[Fundamentals §5](../fundamentals/index.md#5-cap-theorem):

| Property | Meaning |
| -------- | --------- |
| **B**asically Available | The system prioritizes staying available and responsive, even under partition or partial failure |
| **S**oft State | A replica's state can change over time without new input, as it converges with others |
| **E**ventually Consistent | Given no new writes, all replicas eventually converge to the same value — but a read right now isn't guaranteed to see the latest write |

| | ACID | BASE |
| --- | --- | --- |
| Priority | Consistency first | Availability first |
| Consistency | Strong — every read sees the latest committed write | Eventual — reads may be stale until replicas converge |
| Transactions | Full multi-record transactions | Often limited to single-record atomicity, if that |
| Typical fit | Financial balances, inventory counts — anywhere being wrong right now is unacceptable | High-scale, high-availability systems that can tolerate brief staleness |

Not every NoSQL database is purely BASE, and not every relational database
runs at full ACID strictness in practice — this is a spectrum, not a
binary. See
[Distributed Systems §8](../distributed-systems/index.md#8-consistency-models)
for the finer-grained consistency models (strong, eventual, read-after-write,
causal) that sit between these two poles, and which a given NoSQL product
often lets you tune per query.

## 5. NoSQL Database Families

"NoSQL" isn't one thing — it's four distinct families, each optimized for
a different access pattern. The specific product matters less than
recognizing which family it belongs to, since that's what determines
whether it fits your access pattern at all:

| Family | Data model | Optimized for | Example use case | Example Technologies |
| ------ | ----------- | --------------- | ------------------ | ----------------------- |
| Key-Value | Opaque value behind a key | Simple, extremely fast lookups | Session storage, feature flags | Redis, Memcached, DynamoDB, etcd |
| Document | Semi-structured documents (JSON-like) | Flexible schema, nested data | User profiles, product catalogs | MongoDB, Couchbase, Amazon DocumentDB, Firestore |
| Column-Family (Wide-Column) | Rows with dynamic, sparse columns grouped in column families | Very high write throughput, wide rows | Time-series data, event logging | Cassandra, HBase, Google Bigtable, ScyllaDB |
| Graph | Nodes and edges with properties | Traversing relationships | Social graphs, recommendation engines | Neo4j, Amazon Neptune, ArangoDB |

!!! tip "Quick Mental Model"
    A product's marketing category isn't always its full story. **Redis**
    is key-value at its core but also does pub/sub and streams (see
    [Messaging & Events](../messaging-events/index.md)) and doubles as the
    cache in [Caching](../caching/index.md). **DynamoDB** is key-value/
    document hybrid with tunable consistency. **Cassandra** leans AP by
    default (per [Fundamentals §5](../fundamentals/index.md#5-cap-theorem))
    but supports tunable per-query consistency levels. Know the family's
    trade-offs first — the product's specific knobs are a detail you learn
    once you know which family you need.

Relational databases have the same "family vs. product" distinction:
**PostgreSQL**, **MySQL/MariaDB**, **Oracle**, and **SQL Server** are all
the same relational family from §3 above, differing in licensing,
extensions (e.g. PostgreSQL's native JSON and geospatial support), and
operational ecosystem — not in the fundamental ACID/SQL model. The
[Three-Tier LAMP architecture](../../architecture/three-tier-lamp/architecture.md)
uses MySQL/MariaDB as a concrete example of this family in a real platform.

## 6. Distributing Data: What's Covered Elsewhere

Once a database family is chosen, making it scale and stay available
draws on concepts covered elsewhere in this track rather than repeated
here:

* **Partitioning & sharding** — see
  [Scalability §6](../scalability/index.md#6-scaling-the-database-partitioning-sharding)
  and [§7](../scalability/index.md#7-consistent-hashing)
* **Replication (leader-follower, multi-leader, leaderless) and quorum** —
  see [Distributed Systems §6](../distributed-systems/index.md#6-replication-strategies)
  and [§7](../distributed-systems/index.md#7-quorum)
* **Consistency models** — see
  [Distributed Systems §8](../distributed-systems/index.md#8-consistency-models)

## 7. Choosing a Database: A Decision Framework

Work through these questions in order — later ones are only worth asking
once earlier ones have narrowed the field:

| Question | What it rules in or out |
| -------- | -------------------------- |
| **Access pattern** — point lookups, range scans, full-text, graph traversal? | Immediately narrows to one or two families |
| **Consistency** — must reads always see the latest write? | Strong consistency needs point to relational or a strongly-consistent NoSQL configuration |
| **Scale** — what does [Requirements & Estimation](../requirements-estimation/index.md) say about volume and RPS? | Determines whether single-instance is even viable |
| **Transaction requirements** — multi-record atomic operations? | Most NoSQL databases trade this away for scale; if you need it, that's a strong pull toward relational |
| **Query complexity** — ad hoc queries and joins, or known access paths? | Ad hoc/joins favor relational; known access paths are exactly what key-value/document are optimized for |
| **Availability** — can the system tolerate brief unavailability for consistency, or must it always accept writes? | Maps directly to the CP/AP choice in [Fundamentals §5](../fundamentals/index.md#5-cap-theorem) |
| **Operational complexity** — does the team have the expertise to run this in production? | The "best" database on paper that nobody can operate reliably is the wrong choice |

!!! danger "Common Trap"
    Picking a database because of its reputation for scale ("we might need
    to be the next Twitter") rather than the access pattern and
    consistency requirements actually in front of you. Most systems never
    reach the scale that justifies the operational cost of the NoSQL
    database chosen to prepare for it.

## 8. Common Mistakes

* Choosing NoSQL for scale before the estimation numbers ([Requirements &
  Estimation](../requirements-estimation/index.md)) show single-instance
  relational is actually insufficient
* Modeling a graph-shaped problem (deep relationship traversal) in a
  relational database and fighting recursive joins
* Denormalizing everywhere by default, then paying for it in write
  complexity and data-consistency bugs across duplicated copies
* Assuming "NoSQL" implies eventual consistency, or that "SQL" implies it
  can't scale horizontally — both are configuration choices, not fixed
  properties of the category

## 9. Related Concepts

* [Core Building Blocks](../building-blocks/index.md) — where the database
  sits in the overall request path
* [Scalability](../scalability/index.md) — partitioning, sharding, and
  consistent hashing mechanics
* [Distributed Systems](../distributed-systems/index.md) — replication,
  quorum, and consistency models
* [Caching](../caching/index.md) — reducing load on the database chosen here
