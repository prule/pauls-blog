---
title: "Domain Modelling"
layout:     page
draft: false
description: "Advocates for rich domain models that encapsulate business logic and protect invariants, drawing on Domain-Driven Design principles to create more robust and expressive code."
---

## Domain Modelling and Rich Domain Models

### The Core Idea

Domain modelling is the practice of capturing the concepts, rules, and behaviour of a business problem in code. A **rich domain model** goes further — it places business logic *inside* the domain objects themselves, rather than in surrounding service or use case classes.

The opposite of a rich domain model is an **anaemic domain model** — objects that are little more than bags of getters and setters, with all logic extracted into services. Martin Fowler describes the anaemic model as an anti-pattern: it has the structure of OOP without the substance.

---

### Anaemic vs Rich

**Anaemic:**
```kotlin
// Domain object — just data
class Order {
    var id: OrderId? = null
    var status: String? = null
    var items: MutableList<OrderItem> = mutableListOf()
    var total: BigDecimal = BigDecimal.ZERO
}

// Logic lives somewhere else entirely
class OrderService {
    fun cancel(order: Order) {
        if (order.status != "PENDING") throw IllegalStateException("Cannot cancel")
        order.status = "CANCELLED"
    }

    fun addItem(order: Order, item: OrderItem) {
        order.items.add(item)
        order.total = order.items.sumOf { it.price }
    }
}
```

The `Order` class has no opinion about what can be done to it. Any code anywhere can set `status` to anything. The rules are scattered.

**Rich:**
```kotlin
class Order private constructor(
    val id: OrderId,
    private var status: OrderStatus,
    private val items: MutableList<OrderItem> = mutableListOf()
) {
    val total: BigDecimal get() = items.sumOf { it.price }

    fun addItem(item: OrderItem) {
        check(status == OrderStatus.DRAFT) { "Cannot add items to a $status order" }
        items.add(item)
    }

    fun place() {
        check(items.isNotEmpty()) { "Cannot place an empty order" }
        check(status == OrderStatus.DRAFT) { "Order is already $status" }
        status = OrderStatus.PENDING
    }

    fun cancel() {
        check(status == OrderStatus.PENDING) { "Cannot cancel a $status order" }
        status = OrderStatus.CANCELLED
    }

    companion object {
        fun create(customerId: CustomerId): Order =
            Order(OrderId.generate(), OrderStatus.DRAFT)
    }
}
```

The object protects its own invariants. Invalid transitions are impossible — not just discouraged. The logic and the data are co-located.

---

### Building Blocks of a Domain Model

These concepts come largely from Eric Evans' *Domain-Driven Design*, which is the foundational text for this approach.

**Entities** — objects with a distinct identity that persists over time, independent of their attribute values. Two orders with the same items are still different orders. Identity is the defining characteristic. Entities are typically mutable — they transition through states.

**Value Objects** — objects defined entirely by their attributes, with no identity of their own. Two `Money(49.99, GBP)` instances are interchangeable. Value objects should be immutable. They're an underused tool — wrapping primitives in value objects eliminates entire categories of bug.

```kotlin
// Primitive obsession — easy to mix up
fun transfer(amount: BigDecimal, fromAccount: String, toAccount: String)

// Value objects — impossible to pass arguments in the wrong order
fun transfer(amount: Money, from: AccountId, to: AccountId)
```

**Aggregates** — a cluster of entities and value objects treated as a single unit for the purposes of data changes. One entity is the **aggregate root** — the only entry point for modifications. External code holds references to the root, never to internal members directly.

An `Order` aggregate might contain `OrderItem` entities, a `ShippingAddress` value object, and a `Discount` value object. External code calls methods on `Order`; it never reaches in and mutates an `OrderItem` directly.

**Domain Events** — something meaningful that happened in the domain, expressed as an immutable record. `OrderPlaced`, `PaymentFailed`, `SubscriptionRenewed`. Events are past tense — they record facts. They decouple parts of the system and make business processes explicit.

```kotlin
data class OrderPlaced(
    val orderId: OrderId,
    val customerId: CustomerId,
    val placedAt: Instant,
    val total: Money
)
```

**Domain Services** — when an operation belongs to the domain but doesn't naturally belong to any single entity or value object, it lives in a domain service. A `PricingService` that applies complex discount rules across multiple products is a good candidate. Use sparingly — over-reliance on domain services is often a sign of an anaemic model.

**Repositories** — abstractions for retrieving and persisting aggregates. As covered in Clean Architecture, these are interfaces defined in the domain layer, implemented in infrastructure. The domain shouldn't know or care how aggregates are stored.

---

### Protecting Invariants

A domain invariant is a rule that must always be true. Rich domain models enforce invariants at the point of change, not at the point of use.

Techniques for this:

**Private constructors with factory methods** — prevent construction of invalid objects.
```kotlin
class EmailAddress private constructor(val value: String) {
    companion object {
        fun of(raw: String): EmailAddress {
            require(raw.contains('@')) { "Invalid email: $raw" }
            return EmailAddress(raw.lowercase().trim())
        }
    }
}
```

**Encapsulated collections** — never expose mutable internal collections.
```kotlin
// Bad — caller can bypass all business logic
val items: MutableList<OrderItem>

// Good — mutations go through methods that enforce rules
private val _items: MutableList<OrderItem> = mutableListOf()
val items: List<OrderItem> get() = _items.toList()
```

**Sealed state** — model state transitions explicitly, making illegal states unrepresentable.
```kotlin
sealed class OrderStatus {
    object Draft : OrderStatus()
    object Pending : OrderStatus()
    data class Cancelled(val reason: String) : OrderStatus()
    data class Shipped(val trackingCode: String) : OrderStatus()
}
```

With this model, a `Shipped` order always has a tracking code — it can't exist without one. You don't need a nullable field and a runtime check.

---

### Making Illegal States Unrepresentable

This is the highest form of domain modelling — encoding business rules into the type system so that invalid states cannot be constructed, not merely detected at runtime.

```kotlin
// Nullable fields — illegal state is representable, must be checked everywhere
class Shipment {
    var trackingCode: String? = null   // might be present, might not
    var shippedAt: Instant? = null     // same
    var carrier: String? = null        // same
}

// Rich model — illegal state is unrepresentable
sealed class Shipment {
    data class Pending(val orderId: OrderId) : Shipment()
    data class Dispatched(
        val orderId: OrderId,
        val trackingCode: TrackingCode,
        val carrier: Carrier,
        val shippedAt: Instant
    ) : Shipment()
    data class Delivered(
        val orderId: OrderId,
        val trackingCode: TrackingCode,
        val deliveredAt: Instant
    ) : Shipment()
}
```

A `Dispatched` shipment always has a tracking code, carrier, and timestamp. You cannot create one without them. The compiler enforces the business rule.

---

### Ubiquitous Language

A concept from DDD that underpins everything else. The domain model should use the *exact* terminology of the business domain — not technical synonyms, not abbreviations, not programmer jargon. The same words should appear in conversations with domain experts, in requirements documents, and in the code.

If the business says "place an order" — the method is `place()`, not `submit()`, `confirm()`, or `process()`. If they say "a shipment is dispatched" — the state is `Dispatched`, not `Sent` or `Shipped`.

This matters because:
- It eliminates a translation layer between business intent and code
- Domain experts can read (and sometimes write) the model
- Bugs often hide in terminology mismatches — where the code says one thing and the business means another

---

### Relation to Clean Architecture and TDD

These three ideas compose naturally:

- **Clean Architecture** gives domain models a protected home — the innermost layer, free of framework dependencies
- **TDD** drives domain model design — writing tests against a use case naturally leads you to discover what entities, value objects, and rules you need
- **Rich domain models** make the use case layer thinner — if the domain enforces its own rules, the use case orchestrates rather than validates

A use case in a well-modelled system is often surprisingly short:

```kotlin
class PlaceOrderUseCase(
    private val orders: OrderRepository,
    private val events: DomainEventPublisher
) {
    fun execute(command: PlaceOrderCommand) {
        val order = orders.findById(command.orderId)
            ?: throw OrderNotFound(command.orderId)

        order.place()  // all business rules live here

        orders.save(order)
        events.publish(order.domainEvents())
    }
}
```

The use case doesn't validate anything — the domain does. The use case just coordinates.
