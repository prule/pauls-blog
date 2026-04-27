---
title: "Managing Complexity"
layout:     page
draft: false
---

## Managing Complexity — Keeping Systems Comprehensible

### The Two Kinds of Complexity

Fred Brooks introduced the distinction in his 1986 paper *No Silver Bullet*. It remains one of the most useful ideas in software engineering.

**Essential complexity** is inherent in the problem itself. A tax calculation system is complex because tax law is complex. A trading platform has complex rules because financial markets have complex rules. A healthcare system handles nuanced workflows because healthcare is genuinely nuanced. This complexity cannot be removed — it can only be managed, modelled, and represented as clearly as possible. Fighting essential complexity is fighting the domain.

**Accidental complexity** is complexity introduced by the solution rather than the problem. A poor abstraction that makes simple things hard to express. A framework that requires elaborate ceremony for routine operations. An architecture that forces every change to touch six layers. Unnecessary indirection. Leaking implementation details. Inconsistent naming. This complexity has nothing to do with the problem being solved — it is created by the way the solution was built, and it can be removed.

Most codebases contain both in significant quantities. The discipline is distinguishing between them: accepting and carefully managing the essential complexity of the domain, and relentlessly eliminating the accidental complexity introduced by the solution.

The practical test: if a domain expert — someone who understands the business deeply but knows nothing about code — could read your domain model and recognise the concepts, you are representing essential complexity faithfully. If a new developer spends days understanding the codebase before they can make a simple change, you have accumulated significant accidental complexity.

---

### How Accidental Complexity Accumulates

Accidental complexity rarely arrives all at once. It accumulates through many small decisions, each individually defensible, that compound over time.

**Premature abstraction** — an abstraction introduced before the pattern it represents is fully understood. The abstraction captures the wrong thing, or captures the right thing at the wrong level, and every subsequent use case has to fight it slightly. The fights are small at first; over time they become the dominant cost of working in that area.

**Layers that don't earn their keep** — an extra layer of indirection introduced for reasons that no longer apply. A service class that simply delegates to a repository with no transformation. An interface with exactly one implementation that will never have a second. These layers add cognitive load without adding value.

**Inconsistency** — the same concept named three different ways in different parts of the codebase. The same pattern implemented differently in different modules. Inconsistency forces the reader to hold multiple mental models simultaneously and constantly re-establish which convention applies where.

**Accumulated workarounds** — each one added to avoid touching a hard part of the codebase. The hard part doesn't get easier; the workarounds grow around it like scar tissue, making the original problem progressively harder to address and adding indirection that obscures what the code is doing.

**Cargo-culted patterns** — applying a pattern because it was appropriate somewhere else, or because it sounds sophisticated, without considering whether it fits the current problem. Enterprise patterns applied to simple problems create elaborate machinery for doing straightforward things.

The common thread is that each piece of accidental complexity makes the system slightly harder to understand. The next developer working in the area starts from a slightly higher baseline of confusion. Their changes are slightly more conservative because the system is slightly less legible. Over time the system becomes something people work around rather than with.

---

### The Comprehension Cost

Complexity has a primary cost that is easy to overlook: the cognitive load required to understand the system before changing it. Every unnecessary concept, every leaking abstraction, every inconsistency, every layer that doesn't earn its keep adds to the mental overhead of working in the codebase.

This cost is paid repeatedly. Not once when the complexity is introduced, but every time a developer needs to understand the affected area — to fix a bug, add a feature, review a PR, onboard to the codebase. The cumulative cost over the lifetime of a system is enormous, and it's largely invisible because it appears as slowness and caution rather than as a specific line item.

The goal of managing complexity is reducing this comprehension cost. A comprehensible system is one where a developer can form an accurate mental model of a relevant part quickly, make a change with confidence that they understand its effects, and leave the code in a state that the next developer can understand as easily.

---

### Strategies for Managing Essential Complexity

Since essential complexity can't be removed, the goal is to represent it as faithfully and clearly as possible.

**Model the domain explicitly.** The concepts, rules, and relationships of the domain should be visible in the code. If the business talks about invoices, line items, payment terms, and credit notes, those words should appear in the codebase. If a transition from one state to another is only valid under specific conditions, that rule should be encoded in the domain model — not scattered across validation logic in service classes. The domain model is the primary tool for managing essential complexity; a rich, well-named model makes complex domain rules comprehensible.

**Make the rules visible.** Complex business rules buried in implementation details are harder to understand and harder to verify than rules expressed explicitly. A method called `isEligibleForEarlyPaymentDiscount` that contains the rule is more comprehensible than the same logic scattered across several conditionals in a service class. Named concepts pull complexity into the light.

**Use the ubiquitous language consistently.** When the same concept has multiple names in different parts of the codebase, the reader must continuously translate. Enforcing consistent naming — even when it requires renaming things that already exist — pays ongoing dividends in comprehensibility.

**Separate concerns at the right boundaries.** Essential complexity in one area should not leak into adjacent areas. Order pricing rules are complex; that complexity belongs in the pricing domain and should not bleed into the order fulfilment domain. Clean boundaries keep each area's essential complexity local and manageable.

---

### Strategies for Reducing Accidental Complexity

**Name things well and rename them when they're wrong.** A poorly named class or function forces every reader to maintain a mental translation layer. The right name collapses that layer. Renaming is cheap with good tooling and pays ongoing returns. When a name no longer reflects what something does, change it — the friction of renaming is far smaller than the ongoing cost of a misleading name.

**Delete code that isn't earning its keep.** Dead code, unused abstractions, over-engineered infrastructure for problems that never materialised — these all add to the cognitive surface area of the codebase without contributing anything. Deleting code is an act of clarification. A codebase that is smaller and coherent is easier to understand than one that is larger and hypothetical.

**Flatten unnecessary indirection.** Every layer of indirection requires the reader to follow a chain of references to understand what is happening. Some indirection is valuable — it enables testability, replaceability, and separation of concerns. Indirection that exists for its own sake, or that was once necessary and is no longer, adds noise. The question to ask of every layer: what would be lost if this were removed?

**Be consistent.** Pick a pattern and apply it uniformly. If repositories are the abstraction for data access, don't also have DAOs and data managers. If use cases are named as verbs — `PlaceOrder`, `CancelSubscription` — don't have some named as nouns. Consistency allows the reader to form reliable expectations: once I understand how one part works, I can predict how another part works.

**Prefer simple solutions to clever ones.** A clever solution requires the reader to be clever to understand it. A simple solution can be understood directly. The value of cleverness is almost always outweighed by the ongoing comprehension cost it imposes. This applies to language features, design patterns, and algorithms equally — the simpler thing is usually the right thing unless there is a specific, demonstrable reason otherwise.

---

### Complexity and Architecture

Architectural decisions are complexity decisions. The structure of the system determines where complexity concentrates, how far it propagates, and how hard it is to contain.

**Coupling is the mechanism by which complexity spreads.** A change to one component that requires changes to ten others is the system's complexity structure working against you. Reducing coupling — through dependency inversion, clear interfaces, and respecting layer boundaries — is the primary architectural tool for keeping complexity local. When a component's complexity is self-contained, understanding it doesn't require understanding the whole system.

**The architecture should reflect the domain, not the framework.** A codebase whose top-level structure reads as `controllers`, `services`, `repositories` tells you how it was built but not what it does. A codebase whose top-level structure reads as `orders`, `billing`, `fulfilment`, `inventory` tells you what problem it solves. The second structure makes essential complexity visible at the highest level and lets developers navigate to the right area based on what they know about the business rather than what they know about the framework.

**Distributed systems multiply complexity.** Every service boundary introduces network latency, partial failure, consistency challenges, and operational overhead that don't exist in a single process. Microservices are the right answer to specific scaling and organisational problems; they are the wrong answer to complexity management. A system that is hard to understand as a monolith becomes much harder to understand as fifteen services. Complexity must be addressed at its source — in the design and the domain model — before distribution makes sense.

---

### Complexity as a Team Concern

Individual developers can reduce complexity locally — in a function, a class, a module. But the patterns that produce accidental complexity at scale are team patterns, and addressing them requires team-level practices.

**Code review as a complexity check.** Every PR is an opportunity to ask: does this increase or decrease the comprehensibility of the system? Does the new abstraction earn its existence? Is the naming consistent with the rest of the codebase? Could this be simpler? These questions, asked consistently, shape the system's complexity trajectory over time.

**Shared vocabulary and patterns.** A team that has agreed on how to structure use cases, how to name domain concepts, how to handle errors, and how to approach common problems produces a more consistent codebase than one where each developer applies their own preferences. The agreement matters more than which specific approach is chosen.

**Treating comprehensibility as a first-class quality attribute.** Systems are typically evaluated on correctness, performance, and reliability. Comprehensibility — how easy is it to understand and change this system — is rarely measured explicitly, but it determines the long-term cost of everything else. Making it explicit, asking "is this comprehensible" in review, in retrospectives, and in design discussions, keeps it from being crowded out by more immediately measurable concerns.

---

### The Trajectory Matters More Than the Snapshot

A codebase's current complexity level matters less than its trajectory. A complex system that is becoming more comprehensible over time — where each change pays down a little accidental complexity, where patterns are converging, where domain knowledge is being encoded more explicitly — is a system moving in the right direction. A simple system that is accumulating accidental complexity with each delivery — workarounds growing, consistency eroding, the model drifting from the domain — is one heading toward a crisis.

The practices that keep complexity under control are the same practices that appear throughout this series: TDD pushing toward clear interfaces and small focused units, Clean Architecture keeping concerns separated and dependencies pointing inward, rich domain models making essential complexity explicit and well-named, small PRs making each change visible and reviewable, decomposition skill finding the natural boundaries that keep pieces independent. Each practice contributes to comprehensibility; together they form a system for keeping complexity manageable over the lifetime of a product.

Complexity is not a problem that gets solved. It is a property of the system that gets managed — continuously, deliberately, at every level from the function to the architecture. The teams that manage it well don't do so through a single technique or a one-time refactoring effort. They do so through habits of mind applied consistently: naming things well, deleting things that don't earn their keep, keeping concerns separated, and always asking whether the current solution is as simple as the problem actually requires.
