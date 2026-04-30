---
title: "Tidy First"
description: "A summary of Kent Beck's Tidy First?, exploring the relationship between code structure, human behavior, and economic value."
layout:     page
draft: false
---

## Tidy First?

**By Kent Beck — Creator of Extreme Programming & Pioneer of Test-Driven Development**

{{< notice type="info" title="Amazon Affiliates Link" >}}
Buy on Amazon [Tidy First](https://amzn.to/4cB4oRJ)
{{< /notice >}}

---

### Core Thesis

Before you change the *behaviour* of code, should you first tidy its *structure*? That's the deceptively simple question at the heart of this book. Kent Beck's answer is nuanced: **sometimes yes, sometimes no, and knowing the difference is the mark of a mature engineer.**

But the book is really about something deeper — it's a philosophy of software development that connects **code structure, human behaviour, and economic value** into a unified theory of when and why tidying matters.

> **"Make the change easy, then make the easy change."**

---

### Part 1: Tidyings — The Catalogue

Beck opens with a practical toolkit of small, safe structural improvements. These are **not refactors** in the grand sense — they are tiny, reversible moves that can be done in minutes and reviewed separately from behavioural changes.

---

**Key Tidyings:**

**Guard Clauses**
Replace nested conditionals with early returns to flatten the structure and make the happy path obvious.

```kotlin
// Before
fun process(order: Order?) {
    if (order != null) {
        if (order.isValid()) {
            // actual logic buried here
        }
    }
}

// After
fun process(order: Order?) {
    if (order == null) return
    if (!order.isValid()) return
    // actual logic front and centre
}
```

**Dead Code Removal**
Delete it. Don't comment it out. Version control remembers — your codebase shouldn't carry the weight of things that no longer exist.

**Normalise Symmetries**
When the same thing is done in two different ways across a codebase, pick one way and apply it consistently. Inconsistency is invisible cognitive load.

**New Interface, Old Implementation**
If an existing interface is awkward, write the interface you *wish* existed and delegate to the old one. Migrate callers gradually.

**Chunk Statements**
Group related lines of code together with a blank line between conceptually different chunks. Shockingly effective, shockingly underused.

**Explaining Variables & Explaining Constants**
Extract a complex expression into a named variable — not to reuse it, but purely to *name what it means*.

```kotlin
// Before
if (user.age >= 18 && user.country == "AU" && !user.isBanned) { }

// After
val canAccessContent = user.age >= 18 && user.country == "AU" && !user.isBanned
if (canAccessContent) { }
```

**Extract Helper**
Pull a block of code into a well-named function — even if it's only called once. The name does the communicating.

**Reading Order**
Reorder functions and declarations so the code reads top-to-bottom in the order a reader would naturally want to encounter them.

---

### Part 2: Managing Tidying — The Human Side

This section is about the **social and professional dynamics** of tidying, which turn out to be just as important as the technical ones.

---

**Separate Tidying from Behaviour Changes**
This is Beck's most practical rule. **Never mix structural changes with behavioural changes in the same commit or PR.** Why?

- It makes code review nearly impossible
- It obscures what actually changed and why
- It makes bugs harder to trace
- It trains reviewers to distrust your PRs

Tidying PRs should be so boring they get approved in 30 seconds. Behaviour-change PRs should be where all the scrutiny lives.

**Tidying PRs are their own commits**
Small, named, and reviewable on their own terms. "Extracted helper method for fee calculation" is a complete and honest commit message.

**When NOT to Tidy:**
- When you're never coming back to this code
- When the cost of tidying exceeds the benefit of the clarity gained
- When you're under genuine time pressure and behaviour delivery is what matters
- When the code is already clean enough for the change you need to make

---

**The Rhythm of Tidying:**

Beck describes three natural moments to tidy:

| When | Rationale |
|---|---|
| **Before** the behaviour change | Make the change easy first |
| **After** the behaviour change | Leave the campsite cleaner than you found it |
| **Instead of** a behaviour change | Sometimes structure *is* the problem to solve |

The title — *Tidy First?* — is a question, not a command. Judgement is required.

---

### Part 3: Theory — The Economics of Software Design

This is the most intellectually ambitious section of the book. Beck steps back and asks: **why does any of this matter economically?**

---

**Software Has Two Values:**
1. **Behaviour value** — what the software *does* for users today
2. **Structural value** — the system's capacity to change and deliver value tomorrow

Most organisations only measure and reward the first. Beck argues the second is equally — sometimes more — important, and that treating structure as free or irrelevant is a form of **technical debt accumulation** that eventually makes change prohibitively expensive.

---

**Optionality — The Key Economic Concept**
Beck borrows from financial theory. A **software option** is the ability to make a future change. Clean structure *creates options* — it keeps paths open. Messy structure *destroys options* — it makes future changes expensive or impossible.

> **"Structure is not nice to have. It is the mechanism by which you preserve your ability to respond to what you don't yet know."**

Tidying is not aesthetics. It is **option creation** — and options have real economic value.

---

**Coupling & Cohesion — Restated**
Beck revisits these classic concepts with fresh framing:

- **Coupling** — when changing one thing forces you to change another. Coupling is the primary source of software expense.
- **Cohesion** — when things that change together live together. Cohesion reduces the cost of change.

The goal of tidying is almost always to **reduce coupling** or **increase cohesion** — even when you couldn't name it that in the moment.

---

**The Cost of Change Curve**
Traditional software engineering assumes the cost of change rises steeply over time — hence the emphasis on getting design right upfront. Beck's argument is that **good structure flattens this curve**, keeping the cost of change relatively constant throughout a system's life.

This is the economic justification for everything in the book.

---

**Reversible vs Irreversible Decisions**
Tidy first applies most strongly to **reversible** structural decisions — small moves, easy to undo. For irreversible architectural decisions, the calculus is different and requires more deliberation.

---

### The Connecting Thread

Beck is ultimately making a **humanist argument** wrapped in economics. Code is written by people, read by people, and changed by people. Structure that respects the reader's cognitive load is not just technically superior — it is a form of **professional kindness**.

Tidying is the daily practice through which software engineers express care — for their colleagues, for future maintainers, and for the craft itself.

---

### Key Takeaways

| Common Practice | Beck's Reframe |
|---|---|
| Mix tidying with features | Always separate structural and behavioural changes |
| Tidy everything or nothing | Apply judgement — tidy when the value exceeds the cost |
| Structure is a nice-to-have | Structure is option value — it has real economic worth |
| Refactor in big sessions | Tidy in tiny, continuous, safe steps |
| Clean code is about pride | Clean code is about preserving the ability to change |

---

### The Central Message

> **The question "tidy first?" is really the question "do I value my future self and my teammates enough to spend ten minutes making this easier?"** Most of the time, the answer is yes. The tidyings are small. The commits are separate. The economics are sound. And the craft is better for it.

A short book with a long reach — essential reading for any developer who writes code that other humans will have to live with.
