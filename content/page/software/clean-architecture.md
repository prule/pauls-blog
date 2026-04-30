---
title: "Clean Architecture"
layout:     page
draft: false
description: "A deep dive into Clean Architecture principles, explaining how to organize code so business logic remains central and independent of external frameworks and infrastructure."
---

## Clean Architecture

### The Core Idea

Clean Architecture, articulated by Robert C. Martin (Uncle Bob), is a way of organising code so that **business logic is the centre of the system** and everything else — frameworks, databases, UIs, external services — is a detail that can be swapped out or tested independently.

The central rule is the **Dependency Rule**: source code dependencies can only point *inward*. Outer layers know about inner layers; inner layers know nothing about outer layers.

---

### The Layers

Visualised as concentric rings:

```
┌─────────────────────────────────────────┐
│           Frameworks & Drivers          │  ← Web, DB, UI, external APIs
│   ┌─────────────────────────────────┐   │
│   │      Interface Adapters         │   │  ← Controllers, Presenters, Gateways
│   │   ┌─────────────────────────┐   │   │
│   │   │    Application Logic    │   │   │  ← Use Cases
│   │   │   ┌─────────────────┐   │   │   │
│   │   │   │  Domain/Entity  │   │   │   │  ← Business rules, core models
│   │   │   └─────────────────┘   │   │   │
│   │   └─────────────────────────┘   │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Domain (Entities)** — the innermost ring. Pure business objects and rules that would exist regardless of how the software is built. An `Order`, an `Invoice`, a pricing rule — these encode enterprise-wide logic. They have no imports from frameworks, databases, or anything external. They change only when fundamental business rules change.

**Application Logic (Use Cases)** — orchestrates the domain to fulfil specific application goals. A use case like `PlaceOrder` or `CancelSubscription` coordinates entities, calls repository interfaces, and defines the flow of a single operation. It knows about the domain but nothing about HTTP, SQL, or any delivery mechanism.

**Interface Adapters** — translate between the use case layer and the outside world. Controllers convert HTTP requests into use case inputs. Presenters convert use case outputs into view models. Repository implementations translate between domain objects and database rows. This layer speaks both languages — it's the translation zone.

**Frameworks & Drivers** — the outermost ring. Spring, Hibernate, React, PostgreSQL, Kafka — all live here. These are treated as interchangeable infrastructure. The business logic doesn't depend on them; they depend on the business logic.

---

### The Dependency Rule in Practice

If a use case needs to persist data, it doesn't call a database directly. It calls a repository *interface* defined in the application layer. The actual implementation — SQL, NoSQL, in-memory — lives in the outer layer and is injected in.

```kotlin
// Application layer — defines the contract (points inward to domain)
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
}

// Application layer — use case, depends only on the interface
class PlaceOrderUseCase(
    private val orders: OrderRepository,
    private val inventory: InventoryRepository
) {
    fun execute(command: PlaceOrderCommand): OrderId {
        val order = Order.create(command.customerId, command.items)
        inventory.reserve(order)
        orders.save(order)
        return order.id
    }
}

// Infrastructure layer — concrete implementation (points inward to application)
class PostgresOrderRepository(private val db: DataSource) : OrderRepository {
    override fun findById(id: OrderId): Order? { /* SQL here */ }
    override fun save(order: Order) { /* SQL here */ }
}
```

The use case never mentions PostgreSQL. You can swap it for an in-memory implementation in tests without changing a line of business logic.

---

### Boundaries and Data Crossing Them

At each boundary, data is translated into a form appropriate for the receiving layer. You don't pass a database row into a use case, and you don't pass a domain entity directly to a JSON serialiser.

This typically means separate data structures at each boundary:

- **Command / Request model** — data coming into a use case (e.g. `PlaceOrderCommand`)
- **Domain model** — the entity used internally
- **Result / Response model** — what the use case returns to the adapter
- **View model / DTO** — what the controller sends to the client

This is deliberately verbose. The duplication is the point — each layer's data structure can evolve independently without coupling the layers together.

---

### The Humble Object Pattern

A key technique for testability at boundaries. Anything that's hard to test (UI rendering, database I/O, HTTP calls) is stripped down to the minimum logic possible — a "humble object" — and all the interesting logic is moved into a plain, easily-testable object nearby.

For example, a controller should contain almost no logic — it translates the request and delegates to a use case. The use case contains the logic and is pure Kotlin/Java with no framework dependencies. You can test the use case exhaustively without spinning up an HTTP server.

---

### Relation to Other Patterns

Clean Architecture is the synthesis of several earlier ideas, all sharing the same goal:

| Architecture | Origin | Central idea |
|---|---|---|
| **Hexagonal** (Ports & Adapters) | Alistair Cockburn | Domain at centre, adapters plug into ports |
| **Onion Architecture** | Jeffrey Palermo | Layers around a domain core, DI at the boundary |
| **Clean Architecture** | Robert C. Martin | Explicit layer names, strict dependency rule |

They differ in terminology and detail but are fundamentally the same structural idea. If you understand one, you understand all three. Ports & Adapters is worth knowing by name — a *port* is an interface defined by the application layer, an *adapter* is the outer-layer implementation of that interface.

---

### What It Actually Gives You

**Testability.** The domain and use cases are plain objects with no framework magic. You can instantiate and test them with zero infrastructure — no database, no HTTP server, no Spring context. Test suites are fast and reliable.

**Replaceability.** Switching from REST to GraphQL, or PostgreSQL to MongoDB, or Spring to Ktor, touches only the outermost layer. The business logic is unaffected. This sounds theoretical but matters enormously over a system's lifetime.

**Deferral of decisions.** Fielding's original REST insight and Martin's Clean Architecture share a philosophy — the details should be decided late. You can build and test the entire use case layer before choosing a database or a web framework.

**Screaming architecture.** A well-structured Clean Architecture codebase's package structure should reflect the business domain, not the framework. You should see `order`, `billing`, `inventory` at the top level — not `controllers`, `services`, `repositories`. The architecture *screams* what the system does.

---

### The Cost

Clean Architecture is not free. It introduces indirection, more files, and more translation code than a straightforward layered approach. In a small service with stable requirements it can feel like over-engineering.

The trade-off is worth it when:
- The domain is complex and contains real business rules worth protecting
- The system will be long-lived and requirements will change
- Multiple delivery mechanisms are needed (API + CLI + async consumer)
- Fast, comprehensive testing is a priority

For a thin CRUD service with minimal business logic, a simpler structure is often the honest choice. Clean Architecture earns its complexity when the domain justifies it.
