---
layout: "post"
title: "Kotlin Value Objects Beat Primitives"
date: 2026-05-22T12:00:00+10:00
publishDate: 2026-05-22T12:00:00+10:00
draft: false
tags: ["kotlin", "domain-modelling", "clean-code"]
categories: ["Software Engineering"]
description: "Value objects eliminate an entire class of bugs by making illegal states unrepresentable — here's why you should default to them over primitives."
author: "Paul"
---

Use value objects. Primitives leak implementation details, enable invalid states, and produce meaningless function signatures. A `UserId` is not an `Int`. A `Money` is not a `Double`. The type system is your first line of defence — use it.

## The Problem With Primitives

```kotlin
fun transfer(from: Int, to: Int, amount: Double) { ... }
```

Nothing stops a caller passing the arguments in the wrong order. Nothing prevents a negative amount. Nothing distinguishes a user ID from an account ID. The compiler is silent. The bug ships.

Primitive obsession — using raw types for domain concepts — is a common smell. It looks simple but trades short-term convenience for long-term pain.

## Value Objects Fix This

```kotlin
@JvmInline
value class UserId(val value: Int) {
    init { require(value > 0) { "UserId must be positive" } }
}

@JvmInline
value class AccountId(val value: Int) {
    init { require(value > 0) { "AccountId must be positive" } }
}

@JvmInline
value class Money(val cents: Long) {
    init { require(cents >= 0) { "Money cannot be negative" } }
}

fun transfer(from: AccountId, to: AccountId, amount: Money) { ... }
```

Now:
- `UserId` and `AccountId` are incompatible — wrong-order bugs caught at compile time
- Negative money is impossible — invariant enforced at construction
- The function signature is self-documenting

## Kotlin Value Classes: Zero Runtime Cost

Kotlin's `@JvmInline value class` wraps a single property with no heap allocation at runtime. The JVM sees the underlying type. You get type safety for free.

```kotlin
@JvmInline
value class EmailAddress(val value: String) {
    init {
        require(value.contains('@')) { "Invalid email: $value" }
    }
}
```

Validation lives in one place. Every `EmailAddress` in the system is guaranteed valid from construction.

## Equality and Comparison

Value objects compare by value, not identity — which is what you want for domain concepts.

```kotlin
val a = UserId(42)
val b = UserId(42)
println(a == b) // true
```

No need to override `equals`/`hashCode` — `value class` handles it.

## When To Use Them

Use value objects for any domain concept that:
- Has validation rules
- Could be confused with another primitive of the same type
- Carries meaning beyond its underlying type

Common candidates: IDs, money, measurements, email addresses, phone numbers, percentages, coordinates.

{{< notice type="tip" >}}
Start with value objects for all IDs in a new codebase. The habit pays dividends immediately — function signatures become readable and wrong-order bugs disappear.
{{< /notice >}}

## What About Data Classes?

Use `data class` when the concept has multiple fields or needs to participate in collections as a key. Use `value class` for single-field wrappers where performance matters.

```kotlin
// Multi-field: data class
data class DateRange(val start: LocalDate, val end: LocalDate) {
    init { require(!end.isBefore(start)) { "End must be after start" } }
}

// Single-field wrapper: value class
@JvmInline
value class Percentage(val value: Double) {
    init { require(value in 0.0..100.0) { "Percentage out of range: $value" } }
}
```

## Summary

Primitives are convenient but dangerous for domain modelling. Value objects make illegal states unrepresentable, push validation to the boundary, and produce clearer APIs. In Kotlin, `@JvmInline value class` gives you this with no runtime overhead. Default to value objects for domain concepts — use primitives only when you genuinely mean a raw number, string, or boolean.

{{< notice type="tip" >}}
 [Project Valhalla](https://openjdk.org/projects/valhalla/) is bringing value objects to Java!
{{< /notice >}}
