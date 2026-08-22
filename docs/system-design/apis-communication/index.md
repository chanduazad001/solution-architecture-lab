# APIs & Communication

## 1. What is API & Communication Design?

This page covers the contracts and protocols services use to talk to each
other and to clients: which protocol fits which interaction, and the
patterns (rate limiting, pagination, versioning, idempotency) that keep an
API usable and safe as it's adopted by more callers than originally
anticipated.

## 2. Why It Exists

An API is a promise to every caller, and unlike internal code, you can't
always deploy a breaking change and fix every caller at the same time —
some callers are other teams, other companies, or old mobile app versions
still in the wild. The patterns here exist to let an API evolve, absorb
abuse, and communicate failure without breaking callers that did nothing
wrong.

## 3. REST, gRPC, GraphQL

| | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Payload | JSON over HTTP | Binary (Protocol Buffers) over HTTP/2 | JSON over HTTP |
| Client control over response shape | Fixed per endpoint | Fixed per RPC method | Client specifies exactly the fields it needs |
| Best for | Public APIs, broad client compatibility | Internal service-to-service calls, low latency | Clients with varied, evolving data needs (e.g. multiple frontends) |
| Streaming | Not native (needs WebSockets/SSE) | Native bidirectional streaming | Subscriptions (less common) |
| Tooling maturity | Universal | Strong in backend ecosystems | Strong but adds a query-planning layer |
| Trade-off | Simple, cacheable, over/under-fetching common | Fast and strongly typed, harder to debug/inspect than JSON | Flexible for clients, but a single query can be expensive to resolve and cache |

!!! tip "Quick Mental Model"
    REST exposes resources. gRPC exposes procedures, optimized for
    machine-to-machine speed. GraphQL exposes a queryable graph, optimized
    for client flexibility. Pick based on who's calling and what they
    need, not familiarity alone.

## 4. WebSockets & Server-Sent Events

Both exist because REST's request/response model doesn't fit a server
that needs to push updates to a client without the client asking again:

| | WebSockets | Server-Sent Events (SSE) |
| --- | --- | --- |
| Direction | Bidirectional | Server → client only |
| Protocol | Own protocol over a single TCP connection | Plain HTTP, kept open |
| Best for | Chat, collaborative editing, gaming | Live feeds, notifications, progress updates |
| Complexity | Higher — needs its own connection management | Lower — works over standard HTTP infra |

## 5. Rate Limiting & Throttling

Rate limiting protects a service from being overwhelmed — by abuse, a
buggy client in a retry loop, or simply more legitimate traffic than
provisioned for — by rejecting requests once a caller exceeds a defined
budget.

| Algorithm | How it works | Trade-off |
| --------- | -------------- | ----------- |
| Fixed window | Count requests in a fixed time window (e.g. per minute), reset at the boundary | Simple; allows a burst of 2x the limit right at the window boundary |
| Sliding window | Count requests in a rolling window ending now | Smooths out the boundary-burst problem; more bookkeeping |
| Token bucket | Tokens refill at a steady rate; each request consumes a token | Allows controlled bursts up to the bucket size, then throttles to the refill rate |
| Leaky bucket | Requests queue and are processed at a fixed rate | Smooths bursts into a steady output rate; adds queuing latency |

!!! danger "Common Trap"
    Rate limiting only at the API gateway and assuming that protects
    every downstream service. A single expensive endpoint can still
    overload one backend service even while staying under the gateway's
    overall per-client limit — limits sometimes need to be enforced per
    resource, not just per caller.

## 6. Pagination

Returning an entire collection in one response doesn't scale as the
collection grows:

| Strategy | How it works | Trade-off |
| -------- | -------------- | ----------- |
| Offset-based (`?page=3&size=20`) | Skip N records, return the next batch | Simple; "page 3" can shift or duplicate results if records are inserted/deleted between requests |
| Cursor-based (`?after=<id>`) | Return records after a stable reference point | Consistent under concurrent writes; can't jump to an arbitrary page number |

Cursor-based pagination is generally preferred for large, frequently
changing datasets for exactly the consistency reason above.

## 7. API Versioning

Once an API has external callers, changing a field's meaning or removing
one is a breaking change. Versioning strategies (URL path `/v2/`, a
header, or content negotiation) let a breaking change ship without
breaking existing callers, by letting old and new behavior coexist until
callers migrate.

## 8. Idempotency in APIs

As covered in
[Distributed Systems §9](../distributed-systems/index.md#9-idempotency),
a caller that times out can't tell whether its request succeeded. APIs
that accept a client-supplied idempotency key on non-idempotent operations
(e.g. "charge this card") let a safe retry return the original result
instead of performing the operation twice.

## 9. Retries, Timeouts & Circuit Breakers

Every network call in a multi-service system needs a timeout — an
unbounded wait ties up resources indefinitely when a downstream service
hangs — and a retry policy for transient failures. Doing this well across
an entire service graph (avoiding retry storms, failing fast once a
dependency is clearly down) is covered in depth in
[Resilience](../resilience/index.md).

## 10. Request Correlation

In a system made of multiple services, a single client request often
fans out into many internal calls. A **correlation ID** generated at the
edge and passed through every downstream call ties all of those calls
back to one originating request — essential for debugging, and covered in
depth as part of distributed tracing in
[Observability §4](../observability/index.md#4-distributed-tracing-correlation-ids).

## 11. Common Mistakes

* Choosing GraphQL for its flexibility without a plan for query cost
  limiting — a single expensive nested query can do more damage than any
  REST endpoint
* Offset-based pagination on a frequently-written collection, causing
  users to see duplicate or skipped items while paging
* No idempotency key on payment/order-creation endpoints, so a client
  retry after a timeout can create a duplicate charge or order
* Rate limiting configured only at the edge, missing resource-specific
  limits downstream

## 12. Related Concepts

* [Core Building Blocks](../building-blocks/index.md#8-reverse-proxy-vs-api-gateway) —
  where the API gateway sits and what it's responsible for
* [Distributed Systems](../distributed-systems/index.md) — the partial-failure
  reasoning behind idempotency and retries
* [Messaging & Events](../messaging-events/index.md) — the asynchronous
  alternative when a caller doesn't need to wait for a response at all
* [Security](../security/index.md) — authentication and authorization at
  the API gateway
* [Observability](../observability/index.md) — distributed tracing built on
  the correlation IDs introduced in §10
* [Resilience](../resilience/index.md) — timeouts, retries, and circuit
  breakers in depth
