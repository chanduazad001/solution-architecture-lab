# Availability

## 1. Availability Requirements

The redirect path is the product — a short URL that fails to redirect is
a broken link wherever it was shared, often long after the person who
shared it can fix it. Redirect availability is therefore held to a higher
bar than the create path: an outage on URL creation is an inconvenience;
an outage on redirects breaks every link already in circulation.

---

## 2. Redundancy

Following [Availability & Reliability §4](../../availability-reliability/index.md#4-redundancy-failure-domains),
every tier in [Architecture §2](architecture.md#2-high-level-architecture)
runs at least two instances across separate failure domains: multiple API
instances behind the load balancer, a distributed (not single-node) cache,
and a replicated datastore.

---

## 3. Failure Scenarios

| Failure | Impact | Mitigation |
| ------- | -------- | ------------ |
| One API instance fails | None — load balancer routes around it | Health checks (per [Availability & Reliability §6](../../availability-reliability/index.md#6-failover-health-checks-heartbeats)) remove it from rotation |
| Cache cluster fails | Redirects fall back to the datastore — slower, but still correct | Datastore must be sized to absorb full read traffic temporarily; see note below |
| Datastore primary fails | Writes fail; reads continue to be served from cache and read replicas | Failover to a replica, per [Availability & Reliability §7](../../availability-reliability/index.md#7-replication-for-availability) |
| ID generator fails | New short URLs can't be created; existing redirects are unaffected | The write and read paths' independence (per [Architecture](architecture.md)) means this failure doesn't cascade to redirects |

!!! warning "Production Reality"
    "Fall back to the datastore on a cache failure" only works if the
    datastore is provisioned to survive the full, uncached read load —
    otherwise a cache outage doesn't degrade the system, it takes down
    the datastore too, per the cascading-failure pattern in
    [Resilience §11](../../resilience/index.md#11-retry-storms-cascading-failures).
    A circuit breaker and load shedding on the datastore during a cache
    outage keeps a bad situation from becoming a total one.

---

## 4. Eventually Consistent Click Counts

`click_count` (see [Architecture §4](architecture.md#4-data-model)) is
updated asynchronously, as described in
[Data Flow §4](data-flow.md#4-click-count-update). This is a deliberate
consistency trade-off, per
[Fundamentals §4](../../fundamentals/index.md#4-core-quality-attributes):
the count can lag slightly behind actual click volume, in exchange for
never letting analytics tracking add latency or a failure mode to the
redirect itself. A user-facing count that's a few seconds stale is an
acceptable cost; a redirect that's slower or less reliable because of
analytics tracking is not.

---

## 5. Disaster Recovery

Given the redirect path's availability requirement, an RPO close to zero
is justified for the datastore (losing recently-created short URLs is a
real cost — a link already shared becomes permanently broken), while RTO
can tolerate the time a standard failover takes, since a brief failover
window during a rare primary-datastore failure is preferable to the
operational cost of a zero-RTO design for a probability this low. See
[Availability & Reliability §9](../../availability-reliability/index.md#9-disaster-recovery-rto-rpo)
for the RTO/RPO framework this judgment is based on.

---

## 6. Common Mistakes

* Sizing the datastore only for cached-traffic overflow, not full
  uncached read load, turning a cache outage into a datastore outage
* Coupling click-count updates to the redirect's critical path, adding
  latency and a new failure mode to the one request that must never be slow
* Treating the write path's availability requirement as equal to the read
  path's, when the two have very different real-world costs when down
