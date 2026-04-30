---
title: "Configurable Systems"
layout:     page
draft: false
description: "Discusses the transition from building bespoke solutions to creating configurable frameworks that model the shape of a problem, enabling faster delivery through configuration over code."
---

## Configurable Systems and the Right Abstraction

### The Core Idea

There is a class of problem where the right move is not to write another solution, but to recognise that you've already written several solutions to the same underlying problem and extract the pattern. Instead of building bespoke code repeatedly, you build a framework that models the *shape* of the problem — and then configure it for each specific instance.

The payoff is significant: new instances of the problem become configuration rather than code. The framework is tested once deeply; each configuration gets a shallow test. Developers working with it operate at the level of the domain concept — star schema, workflow, form, pipeline — rather than the level of implementation detail.

The risk is equally significant: abstract too early, or at the wrong level, and you build a framework that fights every use case rather than enabling them.

---

### The Star Schema Example

Consider building reporting or data ingestion features. The naïve path is to build each one fresh — bespoke ETL, bespoke data model, bespoke display logic. The third time you do this, a pattern is visible:

- There is always a central **fact** — an event or measurement (a sale, a session, a transaction)
- There are always **dimensions** — the descriptive axes you slice by (time, geography, product, customer)
- There are always **hierarchies** within dimensions — year → quarter → month → day, country → region → city
- The queries, aggregations, and display logic are structurally identical across all of them

At this point the right move is to stop building instances and build the framework:

```kotlin
data class Dimension(
    val name: String,
    val hierarchy: List<Level>,
    val attributes: List<Attribute>
)

data class Level(
    val name: String,
    val column: String
)

data class Fact(
    val name: String,
    val dimensions: List<Dimension>,
    val measures: List<Measure>
)

data class Measure(
    val name: String,
    val column: String,
    val aggregation: Aggregation  // SUM, AVG, COUNT, etc.
)
```

Now a specific reporting domain becomes configuration:

```kotlin
val salesFact = Fact(
    name = "Sales",
    dimensions = listOf(
        Dimension(
            name = "Time",
            hierarchy = listOf(
                Level("Year", "sale_year"),
                Level("Quarter", "sale_quarter"),
                Level("Month", "sale_month"),
                Level("Day", "sale_date")
            ),
            attributes = emptyList()
        ),
        Dimension(
            name = "Product",
            hierarchy = listOf(
                Level("Category", "product_category"),
                Level("Subcategory", "product_subcategory"),
                Level("Product", "product_name")
            ),
            attributes = listOf(Attribute("SKU", "product_sku"))
        )
    ),
    measures = listOf(
        Measure("Revenue", "sale_amount", Aggregation.SUM),
        Measure("Units Sold", "quantity", Aggregation.SUM),
        Measure("Average Order Value", "sale_amount", Aggregation.AVG)
    )
)
```

The code that ingests, queries, and displays this data never needs to change. A new reporting domain is a new configuration block. A developer reading this configuration immediately understands what it represents — it *reads* as a star schema because it *is* a star schema.

---

### What Makes an Abstraction Right

Not every repeated pattern deserves a framework. The discipline is recognising when the underlying shape is genuinely the same versus when surface similarity conceals important differences.

**The right abstraction has a name.** Star schema, pipeline, workflow, form, event sourcing, publish-subscribe — these are concepts with established meaning. When you can name the pattern you're abstracting, that's a strong signal the abstraction is real and not invented. The name comes from the domain, not from the implementation.

**The right abstraction is stable.** The framework should model something that changes slowly. Facts, dimensions, and hierarchies are a decades-old concept in data warehousing — the abstraction will remain valid as specific configurations come and go. An abstraction built around something that is still evolving tends to get the wrong shape and resist change.

**The right abstraction compresses without losing information.** A good configuration should be shorter and clearer than the equivalent bespoke code, and nothing important should be implicit or hidden. If the configuration is as complex as the code it replaces, or if the framework hides details that occasionally matter, the abstraction is leaking.

**Differences that matter should break the abstraction.** If you find yourself adding escape hatches — `customQueryOverride`, `specialCaseHandler`, `rawSqlFallback` — that is the abstraction telling you something. Either the pattern isn't as uniform as you thought, or the framework needs to be split into two distinct abstractions for two genuinely different shapes.

---

### The Design Library Analogy

The same principle applies at the UI level. A design library isn't a collection of finished screens — it's a vocabulary of composable primitives configured for a specific visual language. A `Button` isn't built fresh for each screen; it's configured with variant, size, and intent. A `DataTable` isn't reimplemented for each data set; it's configured with columns, sorting rules, and row actions.

The benefit is the same as the star schema: new screens are assembled from known components rather than written from scratch. Visual consistency is structural rather than disciplinary — you can't accidentally use the wrong button style if there's only one `Button` component. And changes propagate — updating the design token for `primary-colour` updates every component that uses it.

The discipline is the same too. A component that tries to handle every possible variation through a proliferating set of props becomes harder to use than just writing the thing directly. The right component models one concept well and handles its natural variations cleanly.

---

### The Rule of Three

A useful heuristic: the first time you solve a problem, just solve it. The second time you solve the same problem, solve it again but notice the duplication. The third time, extract the abstraction.

This prevents premature abstraction — the failure mode of building a framework before you understand the full shape of the problem. Two instances aren't enough to know what varies and what's fixed. Three instances usually are.

The star schema example works precisely because data warehousing is a mature, well-understood domain. The pattern has been instantiated thousands of times by thousands of teams — the abstraction is confident because the shape is known. When you're in newer territory, waiting for the third instance before abstracting is a meaningful safeguard.

---

### Configuration vs Code as a Design Signal

When a framework is working well, new instances are expressed as data — structures, DSLs, configuration files — rather than as imperative logic. This is a strong signal that the right level of abstraction has been found.

```kotlin
// This is data — it describes what, not how
val inventoryReport = Fact(
    name = "Inventory",
    dimensions = listOf(warehouseDimension, productDimension, timeDimension),
    measures = listOf(
        Measure("Stock Level", "quantity_on_hand", Aggregation.SUM),
        Measure("Reorder Events", "reorder_flag", Aggregation.COUNT)
    )
)
```

No business logic. No control flow. No implementation detail. A domain expert could read this and confirm it's correct. The framework turns this description into working software.

When configuration starts acquiring logic — conditionals, loops, computed values — that's the abstraction boundary eroding. Either the framework needs richer primitives to express that intent cleanly, or the use case has genuinely different needs that belong in code rather than configuration.

---

### The Deeper Principle

The goal is to operate at the level of *what* rather than *how*. Bespoke code answers both questions simultaneously — it describes what it's doing while it does it. A well-designed framework separates them: the configuration says what, the framework knows how.

This is the same separation that makes Clean Architecture valuable, that makes TDD productive, and that makes rich domain models easier to reason about. At every level — architecture, design, implementation — the discipline is finding the right concepts, naming them well, and building systems that speak in those terms rather than leaking their implementation details into every layer.
