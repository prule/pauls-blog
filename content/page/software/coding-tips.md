---
title: "Coding Tips — Principles in Practice"
layout:     page
draft: false
description: "A collection of practical micro-decisions and coding patterns to improve readability, maintainability, and clarity in everyday development."
---

## Coding Tips — Principles in Practice

These tips are the everyday expression of everything covered in this series — the micro-decisions that, made consistently, produce code that is comprehensible, maintainable, and a pleasure to work in.

---

### Control Flow

**Return early to reduce nesting.**
Every level of nesting is a level of context the reader must hold in working memory. Guard clauses that return or throw early eliminate the nesting that builds up around validation and precondition checks.

```kotlin
// Deeply nested — reader must track all conditions simultaneously
fun processOrder(order: Order?): Result {
    if (order != null) {
        if (order.items.isNotEmpty()) {
            if (order.customer.isVerified) {
                // actual logic buried here
            } else {
                throw CustomerNotVerifiedException()
            }
        } else {
            throw EmptyOrderException()
        }
    } else {
        throw OrderNotFoundException()
    }
}

// Early returns — actual logic at the top level, unencumbered
fun processOrder(order: Order?): Result {
    requireNotNull(order) { "Order not found" }
    require(order.items.isNotEmpty()) { "Order has no items" }
    require(order.customer.isVerified) { "Customer is not verified" }

    // actual logic here, at the top level
}
```

**Avoid else after return.**
If the if-branch returns, the else is redundant. Removing it reduces nesting and makes the early-return pattern explicit.

```kotlin
// Redundant else
fun describe(status: OrderStatus): String {
    if (status == OrderStatus.PENDING) {
        return "Awaiting payment"
    } else {
        return "Order is active"
    }
}

// Cleaner without else
fun describe(status: OrderStatus): String {
    if (status == OrderStatus.PENDING) return "Awaiting payment"
    return "Order is active"
}
```

**Prefer positive conditions.**
Negative conditions — `if (!isNotValid)` — require the reader to negate a negation. Name booleans positively and invert the condition when it aids clarity.

```kotlin
// Double negative — reader has to parse the negation
if (!isNotEligible) { applyDiscount() }

// Clear intent
if (isEligible) { applyDiscount() }
```

**Exhaust sealed classes and enums rather than using else.**
An else branch on a when expression over a sealed type silently handles future cases you haven't thought about. Exhaustive branches make the compiler tell you when a new case needs handling.

```kotlin
// else silently swallows new states
return when (status) {
    OrderStatus.PENDING -> "Pending"
    OrderStatus.SHIPPED -> "Shipped"
    else -> "Unknown"  // new states added later will fall here silently
}

// Exhaustive — compiler fails when a new state is added
return when (status) {
    OrderStatus.PENDING -> "Pending"
    OrderStatus.SHIPPED -> "Shipped"
    OrderStatus.CANCELLED -> "Cancelled"
    OrderStatus.DELIVERED -> "Delivered"
}
```

---

### Naming

**Name booleans as questions.**
A boolean that reads as a question answers itself at the call site.

```kotlin
// Ambiguous
if (user.verified) { }
if (order.discount) { }

// Reads naturally
if (user.isVerified) { }
if (order.hasDiscount) { }
```

**Name functions after what they do, not how they do it.**
Implementation details change; intent doesn't. A function named `fetchUserFromDatabase` is coupled to its implementation. `findUser` survives a move to a cache or an API.

**Name variables after what they represent, not their type.**
`val list = listOf(...)` tells you nothing. `val pendingOrders = listOf(...)` tells you everything.

**Make the unit part of the name for numeric values.**
A number without a unit is an accident waiting to happen.

```kotlin
// What is 30? Seconds? Minutes? Days?
val timeout = 30

// Unambiguous
val timeoutSeconds = 30
// Or better — use a typed duration
val timeout = Duration.ofSeconds(30)
```

**Don't abbreviate unless the abbreviation is universal.**
`usr`, `ord`, `cfg`, `mgr` — these save keystrokes at the cost of readability. `user`, `order`, `config`, `manager` are unambiguous. The exception is abbreviations so universal they are clearer than the full word: `id`, `url`, `html`, `dto`.

---

### Functions

**Do one thing at one level of abstraction.**
A function that does two things should usually be two functions. A function that mixes high-level orchestration with low-level detail should have the detail extracted into named helpers.

**Keep the happy path at the top level.**
Validation, guards, and error handling at the top of the function, main logic at the bottom. The reader who wants to understand what the function does — not what it rejects — should be able to skip the guard clauses and read the substance directly.

**Limit parameters.**
A function with many parameters is often doing too much, or has an unextracted concept among its arguments. Three parameters is a soft limit; four is a signal to look for a concept waiting to be named.

```kotlin
// Four parameters — is there a concept here?
fun sendEmail(recipient: String, subject: String, body: String, replyTo: String)

// Extracted concept — the email itself
fun send(email: Email)

data class Email(
    val recipient: EmailAddress,
    val subject: String,
    val body: String,
    val replyTo: EmailAddress
)
```

**Avoid boolean parameters.**
A boolean parameter is often a sign that the function does two things — one for true, one for false. Two well-named functions are almost always clearer.

```kotlin
// What does true mean at the call site?
renderOrder(order, true)

// Intent is visible
renderOrderWithDetails(order)
renderOrderSummary(order)
```

**Return the result rather than mutating a parameter.**
Functions that mutate their inputs have hidden effects that the caller must know about. Returning a new value makes the transformation explicit.

---

### Data and Types

**Avoid primitive obsession.**
A `String` for an email address, an `Int` for a user ID, a `Double` for a monetary amount — these types carry no domain meaning and allow invalid values to be constructed and passed around freely. Wrapping primitives in value objects makes invalid states unrepresentable and makes function signatures self-documenting.

```kotlin
// Three strings — any can be passed in the wrong position
fun transfer(fromAccount: String, toAccount: String, reference: String)

// Three distinct types — passing in the wrong order is a compile error
fun transfer(from: AccountId, to: AccountId, reference: TransferReference)
```

**Use the type system to encode constraints.**
A non-nullable type is a guarantee that a value is present. A sealed class is a guarantee that only known states exist. A value object with validation in its constructor is a guarantee that the value is valid. Each guarantee encoded in the type system is a check that doesn't need to be written — or forgotten — elsewhere.

**Prefer immutability by default.**
A value that cannot change cannot be changed unexpectedly. Immutable data is easier to reason about, safe to share across threads, and simpler to test. Reach for `val` before `var`, immutable collections before mutable ones, and data classes before mutable objects.

**Make null meaningful or eliminate it.**
A nullable type should represent a meaningful absence — a value that legitimately might not exist. Using null to represent errors, uninitialised state, or default values creates ambiguity about what null means. If null is returned from a function, the caller must know which of these it represents.

---

### Classes and Objects

**Keep constructors simple.**
A constructor that does significant work — making network calls, reading files, performing complex computation — is surprising and untestable. Constructors should initialise the object to a valid state and nothing more.

**Expose the minimum necessary.**
Every public method and property is a commitment — an API that callers can depend on. The larger the public surface, the harder the class is to change. Default to private; promote to internal or public only when there is a specific reason to do so.

**Prefer composition over inheritance.**
Inheritance creates tight coupling between parent and child. A change to the parent can affect all children in ways that are hard to anticipate. Composition — holding a collaborator rather than extending it — is more flexible and more explicit about the dependency.

**Don't expose internal collections directly.**
A public mutable collection is a hole in the class's encapsulation. Any caller can modify it, bypassing the class's logic. Return immutable views or copies, and mutate internal collections only through methods that enforce the class's rules.

```kotlin
// Encapsulation broken — caller can add items directly
val items: MutableList<OrderItem> = mutableListOf()

// Encapsulation preserved — mutation goes through a method
private val _items: MutableList<OrderItem> = mutableListOf()
val items: List<OrderItem> get() = _items.toList()

fun addItem(item: OrderItem) {
    check(status == OrderStatus.DRAFT) { "Cannot add items to a $status order" }
    _items.add(item)
}
```

---

### Error Handling

**Fail fast and fail clearly.**
Validate at the point where invalid input is introduced, not several layers later where the context is gone. An error message that explains what was wrong, what was expected, and ideally what the caller should do instead is worth the extra few characters.

**Use exceptions for exceptional conditions.**
An exception represents something that wasn't supposed to happen — a programming error, an external failure, a violated precondition. Using exceptions for normal control flow — returning a result or indicating an expected absence — conflates two different things and makes error handling ambiguous.

**Don't swallow exceptions silently.**
An empty catch block is a lie — it tells the runtime that the error has been handled when it hasn't. At minimum, log the exception with context. Usually, either handle it meaningfully or let it propagate to a layer that can.

```kotlin
// Silent swallow — the error disappears
try {
    processPayment(order)
} catch (e: Exception) { }

// At minimum, log it
try {
    processPayment(order)
} catch (e: PaymentGatewayException) {
    logger.error("Payment processing failed") {
        field("orderId", order.id)
        field("error", e.message)
    }
    throw PaymentFailedException(order.id, e)
}
```

**Catch specific exceptions, not broad ones.**
Catching `Exception` catches everything — including errors you didn't anticipate and shouldn't be handling at this level. Catching the specific exception you expect handles what you intend and lets unexpected exceptions propagate to where they can be dealt with appropriately.

---

### Tests

**Name tests as specifications.**
A test name that describes the behaviour — `when an order is placed with an out-of-stock item, the order is rejected` — is documentation. A test name that describes the implementation — `testPlaceOrderOutOfStock` — is noise.

**One behaviour per test.**
A test that asserts multiple things simultaneously fails without clearly indicating which behaviour failed. Each test should assert one thing, with a name that makes it obvious which behaviour is under examination.

**Arrange, Act, Assert — with a blank line between each.**
The three-phase structure of a test should be visually obvious. The setup, the action, and the assertion are three distinct things; blank lines make the structure explicit and make the test faster to read.

```kotlin
@Test
fun `placing an order reduces available stock`() {
    val product = Product(id = ProductId("p1"), stock = 10)
    val order = Order.create(customerId, listOf(OrderItem(product.id, quantity = 3)))

    inventory.reserve(order)

    assertEquals(7, inventory.availableStock(product.id))
}
```

**Test behaviour, not implementation.**
A test that reaches into private state, mocks internal collaborators, or asserts on intermediate values is testing how the code works rather than what it does. When the implementation changes — without the behaviour changing — these tests break. Tests written against the public interface survive refactoring.

**Make test data communicate intent.**
Use meaningful values in test data rather than arbitrary ones. `customerId = CustomerId("customer-123")` tells the reader nothing. `val premiumCustomer = Customer(tier = Tier.PREMIUM)` tells them exactly which aspect of the customer is relevant to the test.

---

### General Discipline

**Leave the code better than you found it.**
Every time you work in an area, make one small improvement — a better name, an extracted constant, a clarifying comment, a removed TODO. These compound over time without requiring dedicated refactoring sessions.

**Delete code you're not using.**
Unused code is noise that every reader must evaluate and dismiss. Dead code paths, commented-out blocks, unused parameters, unreachable branches — remove them. Version control preserves them if they're ever needed again; the codebase should not.

**Extract constants for magic values.**
A value that appears directly in code without a name forces the reader to infer its meaning from context. Extracting it with a descriptive name collapses that inference into a single readable declaration.

```kotlin
// What is 86400?
if (sessionAge > 86400) expireSession()

// Immediately clear
val SESSION_TIMEOUT_SECONDS = 86400
if (sessionAge > SESSION_TIMEOUT_SECONDS) expireSession()
```

**Treat compiler warnings as errors.**
A warning the team has learned to ignore is a warning that stops providing signal. Warnings that are genuinely not relevant should be suppressed with explanation; warnings that are relevant should be fixed. A codebase where the build produces fifty warnings trains developers to dismiss them all — including the ones that matter.

**Keep the change you're making separate from the cleanup you're doing.**
A PR that mixes a feature change with a large refactor makes both harder to review. If cleanup is needed before or alongside a change, consider separating them: one PR that cleans up, one PR that changes behaviour. The reviewer can verify each independently.

**Read your diff before opening a PR.**
The best code review of your own work happens before the PR is opened. Reading the diff as a reviewer — not as the author who knows what they intended — reveals the things that need a comment, the names that aren't clear to someone without context, the structure that made sense while writing but looks odd from the outside.

---

Leaving comments on your own PR before anyone else reviews it serves several purposes that aren't immediately obvious.

**It distinguishes explanation from improvement.** There are two reasons to comment on your own PR. The first is to explain something genuinely non-obvious — a constraint, a trade-off, a decision that looks wrong but isn't. The second is noticing that the code itself needs improving. The discipline is being honest about which one you're doing. If you find yourself writing a long comment explaining why the code is structured a certain way, the better question is whether the code can be restructured to not need the explanation. A comment that says "I know this looks odd, but..." is sometimes the right move — and sometimes a sign that the code should be clearer.

**It sets the frame for the reviewer.** A reviewer approaching a PR cold has no context about what you were thinking, what alternatives you considered, or what parts you're uncertain about. A well-placed author comment gives them the context to review effectively rather than spending their time reconstructing decisions you've already made. "I considered doing this via X but chose Y because Z" written by the author takes thirty seconds; reconstructed by the reviewer it might take ten minutes — or might not happen at all.

**It identifies the parts you're uncertain about.** Marking the areas where you'd specifically like feedback — "not sure if this is the right abstraction", "this felt clunky but I couldn't find a cleaner way" — focuses the review on the places where it will be most valuable. It also signals intellectual honesty, which tends to produce better review conversations than a PR that presents everything as confident and resolved.

**It prevents defensive conversations.** A non-obvious decision that isn't explained tends to get questioned in review — which can feel like criticism even when it's just confusion. An author comment that pre-emptively explains the reasoning converts that potential friction into a simple acknowledgement. The reviewer understands, the conversation moves on, and nobody has to defend anything.

The one thing to watch is using author comments as a substitute for clarity in the code itself. If every complex part of the PR has an author comment explaining it, that's a signal that the code is carrying too much implicit complexity. The comments should be occasional — for the genuinely non-obvious things — not a running commentary that compensates for code that doesn't communicate well on its own.

---

### Scope and Locality

**Declare variables as close to their use as possible.**
A variable declared at the top of a function and used fifteen lines later forces the reader to hold it in working memory across intervening code. Declaring it immediately before use reduces the span of attention required.

**Keep the scope of variables as narrow as possible.**
A variable that only needs to exist for three lines shouldn't exist for thirty. Smaller scope means fewer places where the value can be misused, and fewer things the reader needs to track simultaneously.

**Avoid reusing variables for different purposes.**
A variable that gets reassigned to hold a different conceptual value at a different point in a function is two variables sharing a name. Give them separate names. The saving in keystrokes is not worth the confusion about what the variable holds at any given point.

---

### Collections and Iteration

**Use the right collection operation for the intent.**
`filter`, `map`, `any`, `all`, `none`, `first`, `find` — each communicates intent directly. A `for` loop that manually builds a filtered list is doing the same thing as `filter` but requiring the reader to reconstruct that intent from the mechanics.

```kotlin
// Mechanical — reader must reconstruct the intent
val result = mutableListOf<Order>()
for (order in orders) {
    if (order.status == OrderStatus.PENDING) {
        result.add(order)
    }
}

// Intentional — intent is immediate
val pendingOrders = orders.filter { it.status == OrderStatus.PENDING }
```

**Name the collection and its elements consistently.**
If the collection is `orders`, the element in a lambda should be `order`. If the collection is `items`, the element should be `item`. Naming the element `it` is fine for very short lambdas where the type is obvious; for anything more complex, a named parameter makes the code significantly clearer.

```kotlin
// it is ambiguous when the lambda is more than one line
orders.filter { it.status == OrderStatus.PENDING && it.total > Money.of(100) }

// Named parameter is clearer
orders.filter { order -> order.status == OrderStatus.PENDING && order.total > Money.of(100) }
```

**Be careful with side effects inside collection operations.**
`map`, `filter`, and `forEach` are conceptually pure transformations. Performing side effects — logging, mutating external state, making network calls — inside them is surprising and hard to reason about. If a side effect is needed, make it explicit and separate from the transformation.

---

### Conditionals

**Extract complex conditions into named booleans.**
A complex conditional is a concept waiting to be named. Extracting it makes the condition readable and the concept reusable.

```kotlin
// Complex inline condition — reader must parse the logic to understand the intent
if (order.status == OrderStatus.PENDING && 
    order.total > Money.of(100) && 
    order.customer.tier == CustomerTier.PREMIUM) {
    applyDiscount()
}

// Named concept — intent is immediate
val isEligibleForPremiumDiscount = order.status == OrderStatus.PENDING
    && order.total > Money.of(100)
    && order.customer.tier == CustomerTier.PREMIUM

if (isEligibleForPremiumDiscount) applyDiscount()
```

**Put the more interesting or specific case first.**
When an if-else handles a specific case and a general case, put the specific case in the if-branch and the general case in the else. The reader's attention is highest at the top; the interesting case should be there.

**Replace repeated if-else chains with a map or a strategy.**
A chain of if-else or when branches that dispatches to different behaviour based on a value is often better expressed as a map from value to behaviour, or as a sealed type with polymorphic dispatch. The chain grows with each new case; the map or the sealed type stays clean.

---

### State and Mutation

**Minimise the distance between where state is created and where it is used.**
State that is created early and modified repeatedly across a long function is hard to reason about because the reader must track its value through each modification. Creating state close to where it is used and minimising modifications reduces the mental overhead of tracking it.

**Prefer transformations to mutations.**
Instead of modifying an object in place, create a new object with the modified value. Transformations are easier to reason about, easier to test, and compose naturally.

```kotlin
// Mutation — order is modified in place, caller's reference changes
fun applyDiscount(order: Order, discount: Discount) {
    order.total = order.total - discount.amount
    order.discountApplied = true
}

// Transformation — original is unchanged, new value is explicit
fun applyDiscount(order: Order, discount: Discount): Order {
    return order.copy(
        total = order.total - discount.amount,
        discountApplied = true
    )
}
```

**Initialise objects completely before sharing them.**
An object that is partially constructed and then passed around, with construction completed by the receiver, is in an invalid intermediate state. Ensure objects are fully valid from the moment they are created.

---

### Dependencies and Coupling

**Depend on abstractions, not concretions, at boundaries.**
Inside a tightly cohesive unit — a single class, a single module — depending directly on concretions is fine and keeps things simple. At boundaries between components, depending on an interface rather than an implementation preserves flexibility and testability.

**Don't reach across layer boundaries.**
A domain object that imports a framework class, a use case that knows about HTTP status codes, a controller that contains business logic — each of these is a layer boundary violation that creates coupling where there should be separation. Each layer should speak only in its own terms.

**Pass in what you need, don't go looking for it.**
A function that receives its dependencies as parameters is testable and explicit. A function that reaches into a global registry, a service locator, or a static accessor to find what it needs hides its dependencies and makes it hard to test and reason about.

---

### Symmetry and Consistency

**If you do something in one place, do it the same way everywhere.**
Inconsistency forces the reader to reason about whether two different approaches are intentionally different or accidentally different. Consistent patterns allow the reader to recognise rather than reason — which is significantly cheaper cognitively.

**Symmetric operations should look symmetric.**
Code that creates a resource and code that destroys it, code that opens a connection and code that closes it, code that increments a counter and code that decrements it — these should live close together and be structured similarly. Asymmetry that isn't justified by a genuine difference is confusion waiting to happen.

**If two things look the same, they should be the same.**
Copy-pasted code that has evolved to be slightly different in two places — one line changed here, one condition added there — is worse than two explicitly different implementations. If the differences are real, name them and make them explicit. If they aren't real, consolidate.

---

### A Few Miscellaneous Ones

**Don't comment out code during debugging and forget to remove it.**
Commented-out debug logging, temporarily disabled assertions, and bypassed validations left in after debugging are noise at best and dangerous at worst. If the debug code was useful enough to write, clean it up before committing; if it will be needed again, make it a proper logging statement.

**Avoid action at a distance.**
Code that changes behaviour elsewhere in the system through a non-obvious mechanism — a mutable global, a thread-local, a shared static — creates the hardest bugs to diagnose. The effect is distant from the cause and the connection is invisible to the reader.

**Be consistent with the level of defensive programming.**
Either validate inputs or trust them — but be consistent. A codebase that sometimes validates, sometimes trusts, and sometimes does both creates uncertainty about what is guaranteed at any given point. The domain boundary is the right place for validation; inside a well-bounded domain, trust the invariants you've established.

**When in doubt, make it boring.**
The most maintainable code is code that does obvious things obviously. Unusual patterns, clever tricks, and non-standard approaches all impose a comprehension tax on every future reader. If there is a boring way to do something and an interesting way, the boring way is usually better. Interesting code is interesting to write; boring code is valuable to maintain.
