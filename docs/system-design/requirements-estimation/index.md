# Requirements & Estimation

## 1. What is Requirements & Estimation?

Before any component or diagram, a system design starts by pinning down
two things: **what the system must do** (functional requirements) and
**how big it needs to be** (capacity estimation). Skipping this step is the
most common reason a design goes wrong — every later decision (database
choice, caching strategy, number of servers) depends on numbers established
here.

## 2. Why It Exists

A load balancer, a cache, and a sharded database are all reasonable
choices — for the right scale. Introducing them for a system with 100
users a day is wasted complexity; not introducing them for a system with
100 million users a day is a system that falls over. Estimation is what
tells you which situation you're actually in, before you design anything.

!!! danger "Common Trap"
    Designing "for scale" by default. A system design that reaches for
    sharding, Kafka, and multi-region replication regardless of the stated
    requirements isn't showing depth — it's skipping the estimation step
    that would have told you whether any of that is justified.

## 3. Functional Requirements Checklist

Functional requirements describe user-facing behavior. Capture them as
short, testable statements:

* What actions can a user perform? (create, read, update, delete, search…)
* What are the core entities and their relationships?
* What must happen synchronously vs can happen asynchronously?
* What's explicitly *out of scope*? (stating this prevents scope creep in
  the design itself)

## 4. Non-Functional Requirements Checklist

Non-functional requirements describe operating constraints, and are what
capacity estimation is actually estimating for:

* **Scale** — how many users, how many requests per second
* **Latency** — acceptable response time (often stated as a percentile, e.g. p99 < 200ms)
* **Availability** — required uptime (see [Fundamentals](../fundamentals/index.md))
* **Consistency** — can reads be stale, or must they always reflect the latest write?
* **Durability** — can any data ever be lost?
* **Read/write ratio** — is this read-heavy (a news feed) or write-heavy (a logging pipeline)?

## 5. Estimating Users and Traffic

The standard inputs, and what they're used for:

| Metric | What it tells you |
| ------ | ------------------- |
| Total registered users | Upper bound on data volume |
| DAU / MAU (Daily / Monthly Active Users) | How many users are actually generating load |
| Requests per user per day | Converts active users into request volume |
| Read/write ratio | Whether to optimize for read scaling (caching, replicas) or write scaling (partitioning, queues) |
| Peak-to-average ratio | Traffic is never flat — capacity must be sized for peak, not average |

## 6. Worked Example: Requests Per Second

Show every step — the goal is a defensible number, not a guess.

```text
Assumptions:
  100,000,000 registered users
  10% are daily active           → DAU = 10,000,000
  Each active user makes
  20 requests/day                → Total = 10,000,000 × 20
                                            = 200,000,000 requests/day

Average RPS:
  200,000,000 requests / 86,400 seconds/day
    ≈ 2,315 requests/second (average)

Peak RPS:
  Traffic is rarely flat across 24 hours. A common rule of thumb
  is a peak-to-average factor of 2-3x for consumer traffic.

  Peak RPS ≈ 2,315 × 3
    ≈ 6,945 requests/second (peak)
```

!!! tip "Quick Mental Model"
    Average load tells you the steady-state cost. Peak load tells you
    what you actually have to build for — undersizing for peak is how
    systems fall over during exactly the moments that matter most.

## 7. Worked Example: Storage Estimation

Example: a photo-sharing feature.

```text
Assumptions:
  10,000,000 DAU
  20% of active users upload 1 photo/day → 2,000,000 photos/day
  Average photo size (with metadata)     → 2 MB

Daily storage:
  2,000,000 photos × 2 MB = 4,000,000 MB ≈ 3.8 TB/day

5-year storage (ignoring deletions/compression):
  3.8 TB/day × 365 days × 5 years ≈ 6.9 PB

Replication factor (durability, typically 3x):
  6.9 PB × 3 ≈ 20.7 PB actual storage required
```

Stating the replication factor separately matters: it's a decision made in
[Core Building Blocks](../building-blocks/index.md) (object storage,
database replication), not an estimation detail to bury silently inside
one number.

## 8. Worked Example: Bandwidth Estimation

```text
Using the photo upload numbers above:

Ingress (uploads):
  2,000,000 photos/day × 2 MB
    = 4,000,000 MB/day ≈ 46.3 MB/s average ingress

Egress (assume each photo is viewed 10x on average):
  2,000,000 photos × 10 views × 2 MB
    = 40,000,000 MB/day ≈ 463 MB/s average egress
```

Egress routinely dwarfs ingress in read-heavy systems — this single
number is often the justification for introducing a CDN (see
[Core Building Blocks](../building-blocks/index.md)).

## 9. Read/Write Ratio and Why It Matters

A 10:1 read/write ratio (typical for a social feed) and a 1:10 ratio
(typical for an ingestion/logging pipeline) lead to different designs from
the same building blocks:

| Read-heavy (e.g. 10:1) | Write-heavy (e.g. 1:10) |
| ------------------------ | -------------------------- |
| Add read replicas | Replicas don't help — writes still go to one place |
| Cache aggressively | Caching has limited benefit |
| Denormalize for read speed | Normalize, or batch writes |
| CDN for static/semi-static content | Queue writes to smooth bursts |

## 10. Common Mistakes

* Giving a final number with no visible assumptions or working — anyone
  reviewing the design can't tell if it's reasonable
* Estimating average load and never converting to peak
* Forgetting the replication/durability factor in storage estimates
* Re-deriving numbers inconsistently later in the same design (e.g. a
  different DAU shows up in the caching section)
* Estimating to unnecessary precision — "≈ 7,000 RPS" is more honest and
  more useful than a falsely precise "6,944.4 RPS"

## 11. Related Concepts

* [Fundamentals](../fundamentals/index.md) — the quality attributes these
  estimates are targets for
* [Core Building Blocks](../building-blocks/index.md) — the components
  these numbers determine the need for (when to add a cache, a replica, a
  CDN, a queue)
* [Case Studies](../case-studies/index.md) — the
  [URL Shortener's capacity estimation](../case-studies/url-shortener/index.md#capacity-estimation)
  applies this exact worked-example approach to one real system
