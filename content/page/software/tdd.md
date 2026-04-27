---
title: "Test-Driven Development"
layout:     page
draft: false
---

## Test-Driven Development (TDD)

### The Core Idea

TDD inverts the traditional write-code-then-test workflow. You write a failing test *before* writing the implementation, then write the minimum code to make it pass, then clean up. The tests don't just verify correctness after the fact — they drive the design of the code itself.

---

### The Red-Green-Refactor Cycle

This is the heartbeat of TDD. Every small increment of work follows three steps:

1. **Red** — Write a test for behaviour that doesn't exist yet. Run it. It must fail. If it passes, either the behaviour already exists or the test is wrong.

2. **Green** — Write the *simplest possible* code to make the test pass. Don't over-engineer. Hardcoding a return value is legitimate at this stage if it satisfies the test.

3. **Refactor** — Clean up the code — remove duplication, improve naming, extract abstractions — without changing behaviour. The tests give you a safety net to do this confidently.

Then repeat. Each cycle is typically minutes, not hours.

---

### A Concrete Example

Say you're building an order pricing system. You start with nothing.

**Red:**
```kotlin
@Test
fun `order with no items has zero total`() {
    val order = Order()
    assertEquals(0.toBigDecimal(), order.total())
}
```
This doesn't even compile yet. That's fine — a compilation failure counts as a failing test.

**Green:**
```kotlin
class Order {
    fun total() = BigDecimal.ZERO
}
```
Hardcoded, but it passes.

**Red:**
```kotlin
@Test
fun `order total is sum of item prices`() {
    val order = Order()
    order.addItem(Item("Widget", price = 9.99.toBigDecimal()))
    order.addItem(Item("Gadget", price = 4.99.toBigDecimal()))
    assertEquals(14.98.toBigDecimal(), order.total())
}
```

**Green:**
```kotlin
class Order {
    private val items = mutableListOf<Item>()

    fun addItem(item: Item) { items.add(item) }
    fun total() = items.sumOf { it.price }
}
```

**Refactor:** nothing to clean up yet. Continue with the next behaviour — discounts, tax, empty state edge cases, etc.

---

### What TDD Actually Gives You

**A design tool, not just a testing tool.** Writing the test first forces you to define the public interface before the implementation. You experience your own API as a consumer before you build it, which tends to produce cleaner, more focused interfaces. If a test is painful to write, that's signal — the design probably has a problem (too many dependencies, too much responsibility, awkward coupling).

**Executable specification.** The test suite becomes a precise description of what the system does. New developers can read the tests to understand intent, not just behaviour.

**Confidence to refactor.** This is underrated. A comprehensive test suite means you can aggressively improve internal structure without fear. Without tests, refactoring is high-risk; with them, it's routine. This keeps codebases from calcifying over time.

**Regression prevention.** Once a bug is fixed, a test is written for it. That failure mode is permanently covered.

---

### Common Pitfalls

**Testing implementation, not behaviour.** Tests that assert internal state or call private methods are brittle — they break when you refactor even if behaviour is preserved. Tests should describe *what* the code does, not *how*.

**Writing tests after the fact.** Post-hoc tests are better than no tests, but they don't drive design. They also tend to be written to match the implementation rather than to specify intent, which reduces their value as a design tool.

**Too coarse a cycle.** Writing a large test that requires building many things before it goes green loses the tight feedback loop. Keep cycles small.

**100% coverage as the goal.** Coverage is a side effect of good TDD, not the objective. Chasing a coverage number leads to tests that execute code without asserting anything meaningful.

**Not refactoring.** Skipping the third step accumulates design debt. Green without refactor is incomplete TDD.

---

### The Testing Pyramid

TDD is most naturally applied at the unit level, but the discipline extends across all layers:

```
        /\
       /  \
      / E2E\          Few — slow, brittle, expensive
     /------\
    /Integr- \        Some — test boundaries between components
   / ation    \
  /------------\
 /    Unit      \     Many — fast, isolated, the TDD heartbeat
/________________\
```

- **Unit tests** — single class or function in isolation, dependencies mocked or stubbed. Fast, deterministic, the core of TDD.
- **Integration tests** — multiple real components together (e.g. your service + a real database). Slower, but verify that the parts connect correctly.
- **End-to-end tests** — the full system from the outside. Valuable but expensive; keep them few and focused on critical paths.

TDD typically drives the unit layer. Integration and E2E tests are added deliberately, not as part of every micro-cycle.

---

### Relation to Design Principles

TDD has a natural affinity with several design principles because it makes violations painful:

- **Single Responsibility** — a class with multiple responsibilities is hard to test in isolation; you end up dragging in unrelated dependencies.
- **Dependency Inversion** — to test a class without its real collaborators, you inject abstractions. TDD nudges you toward interfaces naturally.
- **Small, focused interfaces** — if constructing the subject under test requires ten parameters, the test is telling you something.

In this sense TDD acts as continuous design feedback — every test is a code review of your own interface choices.
