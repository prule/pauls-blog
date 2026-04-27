---
title: "Writing Code for the Reader"
layout:     page
draft: false
---

## Writing Code for the Reader

### The Compiler Is Not Your Audience

Code is written once. It is read many times — by the next developer to work in the area, by a colleague reviewing the PR, by you returning to it six months later with the context entirely gone. The compiler will accept almost anything that is syntactically correct. The human reader is far more demanding, and far more important.

Making this shift explicit — from writing code that works to writing code that communicates — changes a lot of small decisions. Not the big architectural decisions, which are covered elsewhere, but the hundreds of micro-decisions made while writing a function: how to name a variable, whether to extract a helper, how much to put on one line, whether to add a blank line between logical sections. Each decision individually is small. Cumulatively they determine whether the next reader understands the code immediately or struggles with it for twenty minutes.

The question to hold in mind while writing is not "does this work" — that's what tests are for. It's "will the next person who reads this understand what I was trying to do, and why?"

---

### One Level of Abstraction Per Function

One of the most specific and teachable techniques for writing readable code is keeping a consistent level of abstraction within a function. A function that mixes high-level orchestration with low-level detail in the same body forces the reader to context-switch between levels of thinking, which is cognitively expensive.

Consider a function that does the following in sequence: calls a use case, maps the result to a response object, sets three specific HTTP headers, serialises the response to JSON, and writes it to the output stream. The first step is high-level — a business operation. The last three are low-level implementation detail. Reading the function requires shifting mental gears repeatedly.

The fix is to extract the low-level detail into well-named helpers, so the top-level function reads entirely at one level:

```kotlin
// Mixed levels — reader must process detail and intent simultaneously
fun handlePlaceOrder(request: HttpRequest, response: HttpResponse) {
    val command = PlaceOrderCommand(
        customerId = request.pathParam("customerId"),
        items = request.body<List<OrderItemDto>>().map { 
            OrderItem(ProductId(it.productId), Quantity(it.quantity)) 
        }
    )
    val orderId = placeOrderUseCase.execute(command)
    response.setHeader("Content-Type", "application/json")
    response.setHeader("Location", "/orders/${orderId.value}")
    response.setStatus(201)
    response.write("""{"orderId": "${orderId.value}"}""")
}

// Consistent level — reader understands intent immediately, detail available below
fun handlePlaceOrder(request: HttpRequest, response: HttpResponse) {
    val command = buildCommand(request)
    val orderId = placeOrderUseCase.execute(command)
    response.sendCreated(orderId)
}
```

The second version can be understood in a glance. Each line operates at the same level — parse the input, execute the business operation, send the response. The detail is available in `buildCommand` and `sendCreated` for anyone who needs it, but it doesn't intrude on the reader trying to understand the flow.

This principle applies recursively. `buildCommand` should itself be at a consistent level. `sendCreated` should be at a consistent level. Each function, read in isolation, should feel coherent — not like a mixture of things happening at different scales.

---

### Structure and Whitespace Carry Meaning

The physical layout of code on the page is not cosmetic. Blank lines, grouping, and visual structure carry meaning that affects how quickly a reader can parse what they're looking at.

A blank line between logical sections of a function is a paragraph break. It tells the reader: this group of lines forms a unit, and something different starts after the gap. A function with no blank lines presents all of its logic at the same visual weight, forcing the reader to identify the structure themselves. A function with blank lines makes the structure visible before the content is read.

```kotlin
// No structure — reader must identify sections while reading content
fun processPayment(order: Order, card: CardDetails): PaymentResult {
    require(order.total > Money.ZERO) { "Order total must be positive" }
    require(card.expiry.isAfter(LocalDate.now())) { "Card has expired" }
    val gatewayRequest = GatewayRequest(order.total, card.token)
    val gatewayResponse = paymentGateway.charge(gatewayRequest)
    if (gatewayResponse.isDeclined) {
        logger.warn("Payment declined") { field("orderId", order.id) }
        return PaymentResult.Declined(gatewayResponse.declineReason)
    }
    logger.info("Payment successful") { field("orderId", order.id) }
    return PaymentResult.Success(gatewayResponse.reference)
}

// With structure — sections are visible before content is read
fun processPayment(order: Order, card: CardDetails): PaymentResult {
    require(order.total > Money.ZERO) { "Order total must be positive" }
    require(card.expiry.isAfter(LocalDate.now())) { "Card has expired" }

    val gatewayRequest = GatewayRequest(order.total, card.token)
    val gatewayResponse = paymentGateway.charge(gatewayRequest)

    if (gatewayResponse.isDeclined) {
        logger.warn("Payment declined") { field("orderId", order.id) }
        return PaymentResult.Declined(gatewayResponse.declineReason)
    }

    logger.info("Payment successful") { field("orderId", order.id) }
    return PaymentResult.Success(gatewayResponse.reference)
}
```

The second version communicates its structure immediately: validate inputs, make the call, handle the outcome. The reader's eye picks up the paragraph breaks before reading a word, which means they approach each section with a correct expectation of what it contains.

---

### Brevity Is Not a Virtue in Itself

There is a tendency, particularly among developers who have grown comfortable with a language's features, to equate conciseness with quality. Dense code that achieves a lot in few characters can feel elegant. It is often harder to read than the more verbose version.

Deeply chained operations require the reader to evaluate the entire chain before understanding the result. Clever one-liners that combine multiple operations save vertical space but consume working memory. Operator overloading and implicit conversions that reduce characters add cognitive overhead for anyone who doesn't immediately recognise the convention.

```kotlin
// Concise but opaque — reader must mentally evaluate the entire chain
val result = orders
    .filter { it.status == OrderStatus.PENDING }
    .groupBy { it.customerId }
    .mapValues { (_, orders) -> orders.sumOf { it.total } }
    .filterValues { it > Money.of(1000) }
    .keys

// More verbose but communicable — intent is legible at each step
val pendingOrders = orders.filter { it.status == OrderStatus.PENDING }
val totalByCustomer = pendingOrders.groupBy { it.customerId }
    .mapValues { (_, orders) -> orders.sumOf { it.total } }
val highValueCustomers = totalByCustomer
    .filterValues { it > Money.of(1000) }
    .keys
```

The second version is longer. It is also much easier to read, debug, and modify. The named intermediate variables carry intent — `pendingOrders` and `totalByCustomer` tell the reader what they are looking at. If a bug appears in this code, the intermediate values can be inspected. If the logic needs to change, each step can be modified independently.

The right question is not "how few lines can I write this in" but "how quickly can the next person understand what this does and why." Sometimes those produce the same answer. Often they don't.

---

### Failing Loudly and Clearly

How code handles errors is part of its readability — not just for the person reading the source, but for the person debugging the system when something goes wrong. An error that fails loudly with a clear, specific message at the right point is a gift to the next developer. An error that fails silently, propagates to the wrong place, or produces a generic message is a cost they will pay with confusion and time.

Failing at the right point means validating at the boundary where the invalid state is introduced, not catching it several layers later where the original context has been lost:

```kotlin
// Fails late — stack trace points to the wrong place, original context is gone
class OrderService {
    fun calculateTotal(order: Order): Money {
        return order.items.sumOf { it.price * it.quantity }  // NullPointerException here
    }
}

// Fails early — clear message, fails where the problem was introduced
class Order private constructor(
    val items: List<OrderItem>
) {
    init {
        require(items.isNotEmpty()) { 
            "Cannot create an order with no items" 
        }
        require(items.all { it.quantity > 0 }) { 
            "All order items must have a positive quantity" 
        }
    }
}
```

The error message itself is part of the code's readability. A message that says what was wrong, what was expected, and ideally what the caller should do differently is more valuable than a message that names the violated condition in technical terms. The audience for error messages is a developer under pressure trying to understand what went wrong. Write for them.

---

### Not Leaving Loose Ends

A codebase is a communication from the developers who built it to the developers who will maintain it. Loose ends in that communication create noise that every reader must evaluate and dismiss — which is a small cost individually but accumulates into a significant overhead across a large codebase read by many people over time.

**Commented-out code** is the most common loose end. It creates uncertainty: was this removed intentionally? Is it a temporary experiment? Does it represent an approach that was tried and abandoned, or a feature that's coming back? The reader cannot know without context that is rarely present. If code is being removed, remove it. Version control preserves it; the comment does not need to. The one exception is a comment that explicitly explains why a piece of code was removed and what replaced it — that is useful context, not noise.

**TODO comments that will never be addressed** are a form of acknowledged debt that has no plan for repayment. A TODO written in the moment of shipping under pressure is understandable. A TODO that has been in the codebase for two years is false signal — it implies the code is incomplete or suboptimal without providing any actionable information. If it's worth doing, it should be a ticket. If it isn't worth doing, the TODO should be removed. A codebase full of stale TODOs trains developers to ignore them — which means the ones that are genuine get ignored too.

**Half-finished refactors** — where part of the codebase uses a new pattern and the rest uses the old one, with no clear plan for completing the migration — create the inconsistency that is particularly costly for cognitive load. The reader encounters both patterns, must determine which is current, and cannot confidently apply either. If a refactor can't be completed in one PR, the migration path should be explicit and the intermediate state as clean as possible.

**Magic numbers and unexplained constants** force the reader to infer meaning from context. A constant named `MAX_RETRY_ATTEMPTS` with value 3 communicates intent. The number 3 appearing directly in the code does not. The extraction is trivial; the benefit to the reader is disproportionate.

---

### Writing the Code You'd Want to Find

The practices in this piece are not rules to follow mechanically. They are expressions of a single underlying commitment: writing code as if the next person to read it matters.

That person might be a colleague joining the team. It might be you in six months with no memory of writing this. It might be someone dealing with an incident at an inconvenient hour, trying to understand quickly whether this code is relevant to the problem. In each case, code that communicates clearly reduces their burden and speeds their work.

The developer who holds this commitment doesn't ask "does this pass the tests" and stop there. They ask "will the next person understand this" and make the small adjustments — the better name, the extracted function, the blank line, the clarifying comment — that convert working code into communicative code. Those adjustments take minutes. Their value compounds over every subsequent reading of the code, by every developer who encounters it, for as long as the system lives.

Writing for the reader is not a separate activity from writing good code. It is what writing good code means.
