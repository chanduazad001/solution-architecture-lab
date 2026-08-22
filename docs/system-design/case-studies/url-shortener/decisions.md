# Architecture Decisions

ADR = Architecture Decision Record.

## 1. Purpose

This document records the major architectural decisions made for the URL
Shortener case study, following the same format used across the
[Architecture](../../../architecture/index.md) track and the general
[Architecture Decision Making](../../decision-making/index.md) topic page.

---

## ADR-001: Use Base62 Encoding of an Auto-Incrementing ID

### Status

Accepted

### Context

Every short URL needs a unique code, generated at creation time, with no
possibility of two different long URLs colliding on the same code.

### Decision

Generate a unique integer ID from a centralized ID generator, then encode
it into a 7-character base62 string as the short code.

```text
Long URL
  |
  v
Request unique ID (125,000,721)
  |
  v
Base62-encode → "2n9x8Rk"
  |
  v
Store (short_code, long_url)
```

### Alternatives Considered

#### Hash the Long URL

```text
Long URL
  |
  v
MD5("https://example.com/...") → truncate to 7 chars
```

Rejected: hash truncation creates real collision probability at this
system's scale (see the [capacity estimation](index.md#capacity-estimation)
— billions of records), requiring collision detection and retry logic on
every write, which the ID-based approach avoids entirely.

#### Pre-Generated Random Code Pool

```text
Background Job
  |
  v
Generate pool of unused random codes
  |
  v
Hand out from pool on request
```

Rejected: adds an operational component (the pool generator and its
refill logic) to avoid a problem — collision — that base62-encoded unique
IDs don't have in the first place.

### Consequences

#### Positive

- No collision possibility — the ID generator guarantees uniqueness
- Codes are short and predictable in length
- No retry logic needed on the write path

#### Negative

- The ID generator becomes a dependency the write path can't function
  without (see [Availability §3](availability.md#3-failure-scenarios))
- Sequential IDs make short codes somewhat predictable/enumerable, which
  is a minor information-disclosure consideration (an attacker could
  guess adjacent codes) — acceptable here since the codes don't gate
  access to anything sensitive by themselves

---

## ADR-002: Cache-Aside Redis Layer for Redirect Lookups

### Status

Accepted

### Context

The [capacity estimation](index.md#capacity-estimation) shows a 100:1
read/write ratio, with peak reads around 11,700/second. Serving every
redirect from the primary datastore would size the datastore for load
that a cache can absorb far more cheaply.

### Decision

Add a distributed cache in front of the datastore, using the cache-aside
strategy described in
[Caching §3](../../caching/index.md#3-cache-strategies): the application
checks the cache first, and populates it on a miss.

### Alternatives Considered

#### No Cache — Serve All Reads from the Datastore

```text
Client
  |
  v
API
  |
  v
Datastore (every read)
```

Rejected: sizing the datastore for the full uncached peak read volume is
a materially more expensive way to satisfy the same latency requirement
than a cache layer.

#### Write-Through Cache

```text
Client
  |
  v
API
  |
  v
Cache (every write goes here first, synchronously to DB)
```

Rejected: this system's writes are rare relative to reads (per the 100:1
ratio), so there's no meaningful benefit to paying write-path latency for
cache consistency that cache-aside already achieves on the read side.

### Consequences

#### Positive

- The overwhelming majority of redirects never touch the datastore
- Datastore can be sized for write volume and durability, not peak read
  throughput

#### Negative

- Introduces the cache-outage failure mode discussed in
  [Availability §3](availability.md#3-failure-scenarios)
- A short code's first request after cache eviction is always a miss,
  slightly slower than steady state

---

## ADR-003: Asynchronous Click-Count Updates

### Status

Accepted

### Context

Tracking how many times each short URL has been visited is a desirable
feature, but the redirect itself is the availability-critical path
identified in [Availability §1](availability.md#1-availability-requirements).

### Decision

Update `click_count` asynchronously, outside the redirect's critical
path, as described in [Data Flow §4](data-flow.md#4-click-count-update).

### Alternatives Considered

#### Synchronous Increment on Every Redirect

```text
Redirect Request
  |
  v
Look up long URL
  |
  v
Increment click_count (blocking)
  |
  v
Return redirect
```

Rejected: couples the redirect's latency and availability to the
click-count write succeeding, for a feature that isn't essential to the
core product function.

### Consequences

#### Positive

- Redirect latency and availability are unaffected by click-count writes
- A slow or failing analytics path never breaks a redirect

#### Negative

- Click counts are eventually consistent, not exact in real time (see
  [Availability §4](availability.md#4-eventually-consistent-click-counts))
- Requires the asynchronous update mechanism to have its own failure
  handling (retry/DLQ, per
  [Messaging & Events §7](../../messaging-events/index.md#7-dead-letter-queues-retry-backoff))
  so undelivered increments are eventually recovered rather than silently
  lost
