---
title: "Observability"
layout:     page
draft: false
description: "Argues that observability should be a design-time concern, explaining how structured logging, correlation IDs, and metrics enable meaningful production insights."
---

## Observability — Understanding Your System in Production

### Why Observability is a Design Concern

Most teams treat observability as something added after the fact — sprinkle some log statements in, wire up a metrics library, declare the system observable. The result is a production system that generates enormous volumes of data while remaining fundamentally opaque when something goes wrong. The logs exist but don't answer the question being asked. The metrics show something is wrong but not where or why. The traces are missing for the one request that caused the incident.

Observability built as an afterthought reflects the system's implementation rather than its behaviour. Observability designed upfront reflects the system's intent — what it is supposed to do, what can go wrong, and what questions an operator will need to answer at two in the morning.

The shift is from "we should probably log this" to "what do we need to know about this system, and how do we ensure that information is available when we need it?" That question, asked at design time, produces a fundamentally different result.

---

### The Three Pillars

Observability is commonly described in terms of three complementary signals. Each answers different questions and each has gaps the others fill.

**Logs** — discrete records of things that happened. An order was placed. A payment failed. A retry was attempted. Logs are the narrative of the system — they describe events in sequence and carry arbitrary context. They are invaluable for understanding what happened in a specific case, particularly when combined with a correlation identifier that links all log entries for a single request. Their weakness is aggregation — answering "how often does this happen" across millions of log entries is expensive and slow compared to a metric.

**Metrics** — numeric measurements over time. Request rate, error rate, latency percentiles, queue depth, memory usage. Metrics are cheap to store, fast to query, and excellent for alerting — you can ask "is the error rate above 1% right now" in milliseconds. Their weakness is that they discard context. A metric tells you the p99 latency is 3 seconds; it doesn't tell you which requests are slow, for which users, or why.

**Traces** — a record of a request's journey through the system, across service boundaries, with timing for each step. A trace shows you that a particular request spent 2ms in the API layer, 1800ms waiting for a database query, and 200ms in a downstream service. Traces answer "why is this slow" and "where is the time going" in ways that neither logs nor metrics can. Their weakness is cost — tracing every request in a high-volume system is expensive, so sampling is typically required.

The three signals are complementary. Metrics tell you something is wrong. Traces tell you where the problem is. Logs tell you what happened in detail. A mature observability practice uses all three together — alert on a metric, drill into traces to find the slow path, read logs to understand the specific events.

---

### Designing for Observability

**Correlation identifiers are non-negotiable.** Every request entering the system should be assigned a unique identifier at the boundary — the API gateway, the message consumer, the scheduler. That identifier should propagate through every subsequent operation: outbound HTTP calls, database queries, queue messages, log entries. Without it, reconstructing the story of a specific request across distributed components is effectively impossible.

In practice this means:

```kotlin
// At the entry point — generate or accept a correlation ID
val correlationId = request.headers["X-Correlation-ID"] 
    ?: CorrelationId.generate()

// Store in a context that propagates through the call stack
CorrelationContext.set(correlationId)

// All log statements pick it up automatically
logger.info("Processing order") 
// → {"correlationId": "abc-123", "message": "Processing order", ...}

// Outbound calls propagate it
httpClient.post(url) {
    header("X-Correlation-ID", CorrelationContext.get())
}
```

**Structured logging over string concatenation.** Log entries should be machine-readable records, not human-readable sentences. A log entry that reads `"Processing order 12345 for customer 67890"` cannot be queried efficiently. A structured log entry with explicit fields can be filtered, aggregated, and correlated:

```kotlin
// Avoid — string concatenation, unqueryable
logger.info("Processing order $orderId for customer $customerId")

// Prefer — structured, queryable
logger.info("Processing order") {
    field("orderId", orderId)
    field("customerId", customerId)
    field("orderTotal", order.total)
    field("itemCount", order.items.size)
}
```

The structured entry can be indexed by any field. You can ask "show me all orders over £500 that failed" without parsing strings.

**Log at boundaries, not everywhere.** A common anti-pattern is logging at every method call, which produces noise without signal. The useful log entries are at system boundaries — when a request enters, when it leaves, when an external dependency is called, when a significant state transition occurs. Internal implementation details are rarely useful in production logs and dilute the signal.

```kotlin
class PlaceOrderUseCase(
    private val orders: OrderRepository,
    private val payment: PaymentGateway,
    private val logger: Logger
) {
    fun execute(command: PlaceOrderCommand): OrderId {
        logger.info("Placing order") {
            field("customerId", command.customerId)
            field("itemCount", command.items.size)
            field("total", command.total)
        }

        val order = Order.create(command.customerId, command.items)
        orders.save(order)

        val result = payment.charge(order.total, command.paymentDetails)
        
        logger.info("Payment processed") {
            field("orderId", order.id)
            field("paymentResult", result.status)
            field("gatewayReference", result.reference)
        }

        return order.id
    }
}
```

**Emit metrics for the things you will alert on.** Every SLO (service level objective) should have a corresponding metric. If you care about order processing latency, emit a timer around the use case. If you care about payment failure rate, emit a counter for each outcome. Design the metrics from the alert backwards — what will you page someone for, and do you have the metric to detect it?

```kotlin
class InstrumentedPlaceOrderUseCase(
    private val delegate: PlaceOrderUseCase,
    private val metrics: MetricsRegistry
) : PlaceOrderUseCase {
    
    override fun execute(command: PlaceOrderCommand): OrderId {
        val timer = metrics.startTimer("order.placement.duration")
        return try {
            val result = delegate.execute(command)
            metrics.increment("order.placement.success")
            result
        } catch (e: PaymentFailedException) {
            metrics.increment("order.placement.payment_failed")
            throw e
        } catch (e: Exception) {
            metrics.increment("order.placement.error")
            throw e
        } finally {
            timer.stop()
        }
    }
}
```

Note the pattern here: instrumentation is separated from business logic using the decorator pattern. The use case doesn't know it's being measured. This is the same dependency inversion principle from Clean Architecture applied to observability — the business logic is pure, and the observability concerns wrap around it.

---

### Where Observability Lives in the Architecture

Clean Architecture gives observability a natural home. The instrumentation belongs in the adapter and infrastructure layers — not in the domain or use case layers. Domain objects and use cases should not import metrics libraries or logging frameworks. They express business behaviour; observability is a cross-cutting concern.

This means instrumenting at the boundaries:

- **Controllers and adapters** — log incoming requests and outgoing responses, emit request metrics
- **Repository implementations** — log queries and their outcomes, emit database timing metrics
- **External service clients** — log calls and responses, emit latency and error rate metrics, propagate correlation IDs
- **Use case wrappers** — emit business-level metrics (order placed, payment failed, subscription renewed)

The domain model itself is observable through its events. A rich domain model that emits domain events — `OrderPlaced`, `PaymentFailed`, `SubscriptionRenewed` — provides natural hooks for both logging and metrics without coupling the domain to any observability infrastructure. The event handler logs the event and updates the metrics; the domain object simply records what happened.

```kotlin
// Domain event — no observability infrastructure
data class PaymentFailed(
    val orderId: OrderId,
    val reason: PaymentFailureReason,
    val attemptedAmount: Money,
    val failedAt: Instant
)

// Event handler — observability lives here
class PaymentFailedHandler(
    private val logger: Logger,
    private val metrics: MetricsRegistry
) {
    fun handle(event: PaymentFailed) {
        logger.warn("Payment failed") {
            field("orderId", event.orderId)
            field("reason", event.reason)
            field("amount", event.attemptedAmount)
        }
        metrics.increment("payment.failed", tag("reason", event.reason))
    }
}
```

---

### Health Checks and Readiness

Observability extends beyond the runtime behaviour of requests to the operational state of the system itself. Two categories of health endpoint serve different purposes and are worth distinguishing:

**Liveness** — is the process running and not deadlocked? A liveness check answers "should this instance be restarted?" It should be cheap, always available, and not dependent on external systems. If the liveness check fails, the orchestrator restarts the instance.

**Readiness** — is this instance ready to receive traffic? A readiness check verifies that dependencies are reachable and the instance is fully initialised. If the readiness check fails, the load balancer stops sending traffic to this instance without restarting it. These are different conditions requiring different responses, and conflating them causes operational problems — an instance that can't reach the database shouldn't be restarted, it should be removed from rotation.

---

### The Observability Mindset in Code Review

Observability as a design concern shows up in code review. When a new use case, service call, or error condition is introduced, the review should ask:

- Will we know when this fails in production?
- Will we know how often it fails?
- Will we be able to find the specific request that caused a problem?
- Will we be able to tell the difference between a slow dependency and a slow implementation?
- If this goes wrong at 2am, what will the on-call engineer see?

These questions, asked consistently at review time, build observability into the system incrementally rather than bolting it on after the fact. A PR that introduces a new external service call without adding latency metrics and error logging is incomplete in the same way a PR that introduces new behaviour without tests is incomplete.

---

### Practical Starting Points

Observability can feel overwhelming as a topic — distributed tracing infrastructure, metrics pipelines, log aggregation platforms. A useful principle is to start with the highest-value signals and add sophistication as the system demands it.

The minimum viable observability stack for most systems:

**Structured logging to standard output** collected by the platform (Kubernetes, ECS, etc.) and indexed by a log aggregation service. No special infrastructure required in the application — just disciplined structured logging with correlation IDs.

**A small set of key metrics** covering the four golden signals for each service: latency, traffic, errors, and saturation. These four metrics will answer most production questions. Add more as specific gaps become apparent, not speculatively.

**Correlation IDs from day one.** This is the one thing that is very hard to retrofit. A system that has been running in production for two years without correlation IDs has millions of log entries that can't be linked to specific requests. Adding it later requires touching every service boundary simultaneously. The cost of adding it at the start is trivial; the cost of adding it later is significant.

Tracing infrastructure can be added when the system is complex enough that logs and metrics alone leave meaningful gaps — typically when you have multiple services making calls to each other and latency problems are hard to attribute. Starting with traces from day one is usually premature; not having them when you need them is genuinely painful.

---

### The Production Empathy Principle

The underlying principle of observability as a design concern is empathy for the person operating the system in production — who may be you at an inconvenient time. Every design decision about what to log, what to measure, and how to structure that information is an investment in that person's ability to understand and resolve problems quickly.

A system that is easy to debug in production didn't get that way by accident. It got that way because the people who built it asked, at every step, "what do we need to know about this, and how will we know it?" That question, made a habit, is what observability as a design concern actually means.
