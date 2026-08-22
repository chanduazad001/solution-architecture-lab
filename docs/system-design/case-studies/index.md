# Case Studies

Each case study applies every earlier topic in this track to one complete
system, following the same discipline as the
[Architecture](../../architecture/index.md) labs: **Overview → Architecture
→ Data Flow → Security → Availability → Decisions**. Where the topic pages
teach a concept in isolation, a case study shows how several of them
combine under one set of real requirements and trade-offs.

---

## Case Studies

| Case Study | Description | Status |
| ----------- | -------------- | -------- |
| [URL Shortener](url-shortener/index.md) | A read-heavy system centered on one core trade-off: fast, highly-available redirects vs. the write path that creates them. Covers short-code generation, cache-aside redirects, and the read/write asymmetry that shapes the whole design. | Active |

---

## Coming Next

Planned additions, in rough order:

* **Rate Limiter** — the algorithms from [APIs & Communication §5](../apis-communication/index.md#5-rate-limiting-throttling) built into a standalone, reusable service
* **Notification System** — fan-out messaging, delivery guarantees, and multiple channels (email/SMS/push)
* **Chat Application** — WebSockets, message ordering, and presence at scale

---

## How to Read a Case Study

Start with its **Overview** for the problem statement, requirements, and
capacity estimation — the numbers that justify everything that follows.
**Architecture** and **Data Flow** cover the design itself; **Security**
and **Availability** cover the concerns layered on top; **Decisions**
records the ADRs for the choices that had a real alternative worth
recording.
