# URL Shortener

## Problem Statement

Design a service that takes a long URL and returns a short one; visiting
the short URL redirects the user to the original long URL. This is a
deliberately compact problem — the interesting design decisions come from
its read/write asymmetry, not from its feature surface.

## Functional Requirements

* A user can submit a long URL and receive a short URL in return
* Visiting a short URL redirects (HTTP 301/302) to the original long URL
* Short URLs are unique — the same long URL may optionally return the
  same short URL for the same user, but two different long URLs must
  never collide on the same short code
* A short URL can optionally expire after a configured time

## Non-Functional Requirements

* **Read/write ratio** — heavily read-skewed; redirects vastly outnumber
  new-URL creations (a typical assumption, used throughout this case
  study, is 100:1)
* **Latency** — redirects must be fast; a slow redirect is a broken
  product experience, not just a performance nitpick
* **Availability** — redirects should remain available even if the write
  path (creating new short URLs) is degraded
* **Uniqueness** — no two long URLs may ever be assigned the same short
  code

## Constraints & Assumptions

* Short codes are 7 characters, base62-encoded (`[a-zA-Z0-9]`), giving
  62⁷ ≈ 3.5 trillion possible codes — comfortably beyond any realistic
  scale for this system
* Custom aliases (user-chosen short codes) are out of scope for this pass
* Analytics (click counts, referrer tracking) are out of scope beyond a
  basic click counter, noted in [Availability](availability.md) as an
  eventually-consistent concern

## Capacity Estimation

Following the worked-example approach from
[Requirements & Estimation](../../requirements-estimation/index.md):

```text
Assumptions:
  100,000,000 short URLs created per month
  Read/write ratio: 100:1

Write volume:
  100,000,000 / (30 days × 86,400 s) ≈ 39 writes/second (average)

Read volume:
  39 writes/second × 100 ≈ 3,900 reads/second (average)
  Peak (3x average, per Requirements & Estimation §6) ≈ 11,700 reads/second

Storage (5 years):
  100,000,000 URLs/month × 60 months = 6,000,000,000 records
  ~500 bytes/record (long URL + short code + metadata)
    = 6,000,000,000 × 500 bytes ≈ 2.8 TB (before replication)
  With a 3x replication factor (see Availability & Reliability):
    ≈ 8.4 TB
```

The read/write asymmetry above is the single number that shapes the rest
of this design — see [Architecture](architecture.md) for how.

## High-Level Approach

```text
Long URL
    ↓
Generate unique short code
    ↓
Store (short code → long URL) mapping
    ↓
Return short URL to user
```

```text
Short URL visited
    ↓
Look up long URL by short code
    ↓
Redirect (HTTP 301/302) to long URL
```

Full component-level detail is in [Architecture](architecture.md); the
request path with cache behavior is in [Data Flow](data-flow.md).

## Technology Stack

| Layer | Technology |
| ----- | ----------- |
| API / Application Tier | Stateless service behind a load balancer |
| Cache | Distributed cache (cache-aside, per [Caching](../../caching/index.md)) |
| Primary Datastore | Key-value or relational store — see [Decisions](decisions.md) |
| Short Code Generation | Base62 encoding of an auto-incrementing ID |

## Related Case Studies

* [Case Studies overview](../index.md) — a planned Rate Limiter case study
  will cover the abuse-prevention concern touched on in
  [Security](security.md) in full depth
