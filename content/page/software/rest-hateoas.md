---
title: "REST and HATEOAS"
layout:     page
draft: false
---

## REST and HATEOAS

### REST (Representational State Transfer)

REST is an architectural style for distributed hypermedia systems, first described by Roy Fielding in his 2000 doctoral dissertation. It's not a protocol or standard, but a set of constraints that, when applied to a web service, produce desirable properties like scalability, simplicity, and modifiability.

**The six constraints of REST:**

1. **Client-Server** — Concerns are separated between the UI/consumer and the data storage/logic. This allows each to evolve independently.

2. **Stateless** — Each request from client to server must contain all information needed to understand and process the request. No session state is stored on the server between requests.

3. **Cacheable** — Responses must define themselves as cacheable or non-cacheable, allowing clients and intermediaries to reuse responses and improve efficiency.

4. **Uniform Interface** — The central feature of REST. It simplifies and decouples the architecture via four sub-constraints:
    - *Resource identification* — Resources are identified by URIs (e.g. `/users/42`)
    - *Manipulation through representations* — Clients interact with resources via representations (JSON, XML, etc.), not directly
    - *Self-descriptive messages* — Each message includes enough information to describe how to process it (e.g. `Content-Type: application/json`)
    - *Hypermedia as the engine of application state* — **HATEOAS** (see below)

5. **Layered System** — The client need not know whether it's connected directly to the server or an intermediary (load balancer, cache, gateway).

6. **Code on Demand** *(optional)* — Servers can extend client functionality by transferring executable code (e.g. JavaScript).

A service that satisfies these constraints is described as RESTful. In practice, most so-called "REST APIs" only partially comply — they typically implement resources and HTTP verbs correctly but omit HATEOAS entirely.

---

### HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS is the most misunderstood and most commonly omitted REST constraint. It states that a client should be able to navigate an entire API dynamically through hypermedia links embedded in responses — with no prior knowledge of the API's URL structure beyond a single entry point.

**The core idea:** a response doesn't just return data — it also returns the set of actions currently available from that state, as links.

**Concrete example — without HATEOAS:**

```json
GET /orders/123

{
  "id": 123,
  "status": "pending",
  "total": 49.99
}
```

The client must have out-of-band knowledge to know it can call `POST /orders/123/cancel` or `POST /orders/123/pay`. If those URLs change, clients break.

**With HATEOAS:**

```json
GET /orders/123

{
  "id": 123,
  "status": "pending",
  "total": 49.99,
  "_links": {
    "self":   { "href": "/orders/123",        "method": "GET"    },
    "pay":    { "href": "/orders/123/payment", "method": "POST"   },
    "cancel": { "href": "/orders/123/cancel",  "method": "DELETE" }
  }
}
```

If the order were already paid, the `pay` link would simply not appear — the client doesn't need to reason about business logic to know what's possible. The server drives valid transitions.

---

**Common hypermedia formats:**

| Format | Description |
|---|---|
| **HAL** (`application/hal+json`) | Simple `_links` and `_embedded` convention; widely used |
| **JSON:API** | Opinionated spec covering links, relationships, pagination, errors |
| **Siren** | Richer format including typed actions with input fields |
| **JSON-LD / Hydra** | Semantic web approach; links resources to vocabularies |

---

That's a great point — and arguably the most *practical* case for HATEOAS in real systems. I'll refine that section:

---

**Why HATEOAS Matters (and Why It's Ignored)**

**The theoretical benefits:**
- Clients are decoupled from URL structure — the server can reorganise its URL space without breaking clients
- Self-documenting at runtime — a client can discover capabilities by following links
- **Server-driven affordances** — the server expresses what actions are *currently valid* for a given resource, based on its state and the authenticated user's permissions. A client doesn't need to replicate the logic of "can this user cancel this order?" — it simply checks whether the `cancel` link is present. This keeps business rules and authorisation logic in one place, reducing drift between server and client.

For example, the same `GET /orders/123` endpoint might return different link sets depending on context:

- An **admin** sees `cancel`, `refund`, and `escalate`
- A **customer** sees only `cancel` (while the order is pending) or no mutation links at all (once shipped)
- A **read-only** role sees `self` only

The client renders its UI based on what the server says is possible — rather than independently re-implementing permission checks that can fall out of sync.

---

This is where HATEOAS has genuine engineering value even in tightly-coupled systems where URL discovery isn't a concern. The link set becomes a server-authorised capability manifest, not just a navigation aid.

---

### Richardson Maturity Model

A useful way to grade how RESTful an API is:

| Level | Name | Characteristic |
|---|---|---|
| 0 | Plain HTTP | Single URI, single method (e.g. RPC-style) |
| 1 | Resources | Multiple URIs representing distinct resources |
| 2 | HTTP Verbs | Correct use of GET, POST, PUT, DELETE, etc. |
| 3 | Hypermedia | HATEOAS — responses include navigable links |

Most APIs ship at level 2 and call it REST. Level 3 is rare but represents the full architectural intent.
