---
title: "Modelling Domains in Kotlin With Value Objects"
layout:     page
draft: false
description: "Demonstrates the power of value objects in Kotlin to eliminate primitive obsession, enforce domain invariants at compile time, and create more expressive, self-documenting code."
---

## Modelling Domains in Kotlin With Value Objects

### The Problem With Primitives

Every domain has concepts that primitives can represent but cannot express. An email address is a `String`, but not every `String` is an email address. An order total is a `BigDecimal`, but not every `BigDecimal` is a valid order total — it shouldn't be negative, it should have a defined currency, and it shouldn't be mixed with a price in a different currency. A user ID is an `Int`, but it shouldn't be added to a product ID, compared to a quantity, or passed where an order ID is expected.

When these concepts are represented as primitives, the type system provides no help. The compiler cannot distinguish a user ID from a product ID — they're both `Int`. It cannot prevent a price in GBP being added to a price in USD — they're both `BigDecimal`. It cannot catch an email address being passed where a username is expected — they're both `String`. These errors are invisible until runtime, and sometimes not even then — they compile, they run, and they produce wrong results silently.

Value objects solve this by giving domain concepts their own types. The type system then enforces what the domain requires — not as runtime checks scattered across the codebase, but as compile-time guarantees that can never be bypassed.

---

### What a Value Object Is

A value object is an immutable type defined entirely by its value. Two value objects with the same value are equal — there is no notion of identity separate from the value itself. A `Money(100, GBP)` and another `Money(100, GBP)` are the same thing. An `OrderId("abc-123")` and another `OrderId("abc-123")` are the same order ID.

In Kotlin, value objects map naturally onto data classes — or, for single-value wrappers, value classes, which have zero runtime overhead:

```kotlin
@JvmInline
value class OrderId(val value: String)

@JvmInline
value class CustomerId(val value: String)

@JvmInline
value class ProductId(val value: String)
```

These three types are each backed by a `String` at runtime, but they are distinct types at compile time. The compiler treats them as different things. Passing an `OrderId` where a `CustomerId` is expected is a compile error.

---

### Preventing Parameter Confusion

The most immediately valuable property of value objects is preventing arguments being passed in the wrong order. This is one of the most common and most silent bugs in codebases that rely on primitives.

```kotlin
// Primitives — three strings, any order is accepted by the compiler
fun sendConfirmation(customerId: String, orderId: String, emailAddress: String)

// Called correctly — but is it? The compiler can't tell
sendConfirmation(customer.id, order.id, customer.email)

// Called incorrectly — compiles without complaint
sendConfirmation(order.id, customer.email, customer.id)
```

With value objects:

```kotlin
fun sendConfirmation(customerId: CustomerId, orderId: OrderId, email: EmailAddress)

// This no longer compiles — the mistake is caught immediately
sendConfirmation(order.id, customer.email, customer.id)
```

The compiler has become a domain expert. It knows that an `OrderId` is not a `CustomerId`, and that neither is an `EmailAddress`. The entire class of argument-ordering bugs becomes impossible.

---

### Encoding Validation Once

With primitives, validation is duplicated. Every function that receives an email address string must either validate it or trust that the caller already has. The validation logic is scattered, inconsistently applied, and easy to forget.

With a value object, validation happens once — in the constructor. Once a value object exists, it is valid by definition. No subsequent code needs to check.

```kotlin
@JvmInline
value class EmailAddress private constructor(val value: String) {
    companion object {
        fun of(raw: String): EmailAddress {
            require(raw.contains('@')) { "Invalid email address: $raw" }
            require(raw.length <= 254) { "Email address too long: $raw" }
            return EmailAddress(raw.lowercase().trim())
        }
    }
}
```

Now every function that receives an `EmailAddress` knows it is valid. The validation rule — what constitutes a valid email — exists in exactly one place. If the rule changes, it changes everywhere simultaneously.

```kotlin
// Before — validation scattered, easy to forget
fun register(email: String, password: String) {
    if (!email.contains('@')) throw InvalidEmailException(email)
    // ... and every other function that takes an email does the same
}

// After — validation centralised, impossible to bypass
fun register(email: EmailAddress, password: Password) {
    // email is guaranteed valid — no check needed
}
```

---

### Making Units Explicit

Numeric primitives carry no unit information. A `Double` representing a duration could be seconds, milliseconds, or microseconds. A `BigDecimal` representing money could be any currency. When these are mixed — which the compiler happily permits — the results are wrong and silent.

```kotlin
// What units are these? The compiler doesn't know or care
fun applyLatency(base: Double, overhead: Double): Double = base + overhead

// Called with mixed units — compiles, runs, produces nonsense
val latency = applyLatency(2.0, 500.0) // 2 seconds + 500 milliseconds = wrong
```

Value objects make units part of the type:

```kotlin
@JvmInline
value class Seconds(val value: Double)

@JvmInline  
value class Milliseconds(val value: Double) {
    fun toSeconds(): Seconds = Seconds(value / 1000)
}

fun applyLatency(base: Seconds, overhead: Seconds): Seconds = 
    Seconds(base.value + overhead.value)

// This no longer compiles — units must be explicit
applyLatency(Seconds(2.0), Milliseconds(500.0))

// Correct — conversion is explicit and visible
applyLatency(Seconds(2.0), Milliseconds(500.0).toSeconds())
```

The Mars Climate Orbiter was lost in 1999 because one system used metric units and another used imperial units, both represented as plain numbers. Unit confusion in software is not hypothetical.

---

### Money and Currency

Money is the canonical example of where primitive representation causes real damage. A `BigDecimal` knows nothing about currency. Adding two `BigDecimal` values representing prices in different currencies produces a number that means nothing.

```kotlin
data class Money(
    val amount: BigDecimal,
    val currency: Currency
) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { 
            "Cannot add $currency and ${other.currency}" 
        }
        return Money(amount + other.amount, currency)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot subtract ${other.currency} from $currency"
        }
        require(amount >= other.amount) {
            "Cannot subtract ${other.amount} from ${amount}: result would be negative"
        }
        return Money(amount - other.amount, currency)
    }

    operator fun times(multiplier: BigDecimal): Money =
        Money(amount * multiplier, currency)

    companion object {
        fun of(amount: Number, currency: Currency): Money =
            Money(BigDecimal(amount.toString()), currency)
        
        val ZERO_GBP = Money(BigDecimal.ZERO, Currency.GBP)
    }
}
```

Now the domain rules about money are encoded in the type. Cross-currency addition fails at the point of the mistake with a clear message. Negative money is impossible. The compiler and the domain object together make an entire class of financial bugs impossible to introduce silently.

```kotlin
// Before — silent wrong result
val total = priceInGBP + priceInUSD  // BigDecimal addition, currency ignored

// After — fails immediately with a clear message
val total = priceInGBP + priceInUSD  // throws: Cannot add GBP and USD
```

---

### Quantities and Counts

A quantity of items is not just an integer. It should not be negative. It should not be a fractional number. It might have a maximum. And it should not be confused with a price, an ID, or a line number.

```kotlin
@JvmInline
value class Quantity private constructor(val value: Int) {
    companion object {
        fun of(value: Int): Quantity {
            require(value > 0) { "Quantity must be positive, was $value" }
            return Quantity(value)
        }
    }

    operator fun plus(other: Quantity): Quantity = Quantity(value + other.value)
    
    operator fun minus(other: Quantity): Quantity {
        require(value >= other.value) { 
            "Cannot subtract ${other.value} from $value: insufficient quantity" 
        }
        return Quantity(value - other.value)
    }

    operator fun compareTo(other: Quantity): Int = value.compareTo(other.value)
}
```

The constraint — quantities must be positive — is expressed once, at construction time, and enforced everywhere. A function that receives a `Quantity` never needs to check if it's negative. A `Quantity` can never be confused with an `OrderId` even though both might be integers.

---

### Identifiers

Every entity in the domain has an identifier. Representing all identifiers as `String` or `Long` makes them interchangeable in the type system — which they are not in the domain.

```kotlin
@JvmInline value class OrderId(val value: String)

@JvmInline value class CustomerId(val value: String)

@JvmInline value class ProductId(val value: String)

@JvmInline value class ShipmentId(val value: String)
```

This costs almost nothing — value classes have zero overhead at runtime. The benefit is that every function signature becomes a precise statement of what it requires:

```kotlin
// Primitives — any string accepted, caller must know the correct order
fun findOrdersForCustomer(customerId: String): List<Order>

// Value objects — intent is precise, mistakes are caught at compile time
fun findOrdersForCustomer(customerId: CustomerId): List<Order>
```

The function signature is now self-documenting. The compiler enforces correct usage. And the refactoring that renames the underlying storage column for customer IDs cannot accidentally cause a product ID to be passed in its place.

---

### Composed Value Objects

Value objects compose naturally into richer domain concepts. Rather than a loose collection of primitives representing an address, the address itself becomes a value object:

```kotlin
data class Address(
    val line1: AddressLine,
    val line2: AddressLine?,
    val city: City,
    val postcode: Postcode,
    val country: Country
)

@JvmInline
value class Postcode private constructor(val value: String) {
    companion object {
        fun of(raw: String): Postcode {
            val normalised = raw.uppercase().replace(" ", "")
            require(normalised.matches(Regex("[A-Z]{1,2}[0-9][0-9A-Z]?[0-9][A-Z]{2}"))) {
                "Invalid UK postcode: $raw"
            }
            return Postcode(normalised)
        }
    }
}
```

Now a `ShippingAddress` and a `BillingAddress` can be different types even though they have the same structure — preventing a shipping address being used where a billing address is required, which is a meaningful domain distinction.

```kotlin
@JvmInline value class ShippingAddress(val address: Address)
@JvmInline value class BillingAddress(val address: Address)

fun processOrder(shipping: ShippingAddress, billing: BillingAddress)

// These are different types — passing them in the wrong order won't compile
processOrder(billingAddress, shippingAddress) // compile error
```

---

### The Ripple Effect Through the Codebase

Introducing value objects doesn't just improve the places where they are defined. The improvement ripples outward through every function signature that uses them, every test that constructs them, and every error message that mentions them.

**Function signatures become precise.** A function that takes `(String, String, BigDecimal, String)` says nothing about its domain. A function that takes `(CustomerId, ProductId, Money, EmailAddress)` is a precise statement of what it needs and what those things mean.

**Tests become clearer.** Test data constructed from value objects communicates which domain concept is being exercised. `CustomerId("customer-vip")` in a test is more meaningful than `"customer-vip"` — and it guarantees the test is passing a valid customer ID, not accidentally passing a product ID.

**Error messages become precise.** When a value object's validation fails, the error message can name the concept — "Invalid postcode: SW1A1AB" — rather than producing a generic "Invalid string" message. The domain concept is present in the error, which makes it immediately actionable.

**Refactoring becomes safer.** When an ID changes its underlying representation — from integer to UUID, from short string to namespaced string — the change is made in one place: the value object. Every caller is still passing an `OrderId`; the internal representation changes without touching any call site.

---

### The Investment

The upfront cost of value objects is real but small. Each wrapper requires a few lines to define, a constructor to validate, and potentially a handful of operations. The ongoing return is a codebase where an entire class of bugs — wrong argument order, invalid values, mixed units, cross-currency arithmetic — is structurally impossible rather than merely unlikely.

The type system stops being an obstacle that must be appeased and starts being a collaborator that enforces domain rules automatically. Every new function written against value-object-typed parameters gets the domain constraints for free. Every refactor that changes underlying representations is contained within the value object. Every test that constructs domain objects through their value objects is guaranteed to start from a valid state.

Primitive obsession is one of the most common and most treatable code smells in real codebases. The treatment is not complex. It is a series of small, deliberate decisions to give domain concepts their own types — and let the compiler do the work of enforcing what the domain requires.
