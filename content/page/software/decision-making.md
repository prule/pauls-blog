---
title: "Decision Making"
layout:     page
draft: false
description: "Explores strategies for making architectural decisions under uncertainty, distinguishing between reversible and irreversible choices to maintain optionality and speed."
---

## Decision Making Under Uncertainty

### The Problem With Deciding Too Early

Architecture decisions are made at the worst possible time. At the start of a project — when the key decisions feel most urgent — the team has the least information. The domain isn't fully understood, the usage patterns are hypothetical, the team hasn't discovered the pain points yet. And yet this is precisely when decisions are made that will shape the system for years.

The instinct to resolve uncertainty by deciding is understandable. Ambiguity is uncomfortable. Stakeholders want a plan. Developers want to start building. So decisions get made, foundations get laid, and the system starts to accumulate the consequences of choices made before the information needed to make them well was available.

The alternative is not to avoid decisions — that's paralysis. It's to understand which decisions need to be made now, which can and should be deferred, and what it costs to get each category wrong.

---

### Type 1 and Type 2 Decisions

Jeff Bezos articulated a distinction at Amazon that is directly applicable to architecture. He described two types of decisions:

**Type 1 decisions** are irreversible or nearly so. They are one-way doors — once you walk through, going back is expensive, disruptive, or effectively impossible. Choosing a cloud provider that your entire infrastructure will depend on. Adopting an event sourcing architecture for your core domain. Designing a public API contract that external systems will depend on. Selecting a database technology that will underpin a system at scale. Getting these wrong is costly because the correction requires dismantling and rebuilding something fundamental.

**Type 2 decisions** are reversible. They are two-way doors — you can walk through, look around, and come back if it turns out to be wrong. The name of an internal class. The structure of a private API. The choice between two libraries that are equally well-encapsulated behind an interface. Whether to extract a service now or keep it in the monolith for another quarter. Getting these wrong is cheap because the correction is straightforward.

Bezos's observation was that large organisations tend to treat Type 2 decisions with the same caution as Type 1 decisions — running them through heavyweight processes, building consensus, deferring to committees — which makes them slow without making them better. The process appropriate for a one-way door is wrong for a two-way door.

The same failure mode appears in software teams. Lengthy architectural debates about things that could be changed cheaply next month. Premature commitment to patterns that haven't been validated. Treating everything as irreversible because the distinction hasn't been drawn. The cost is speed without the benefit of correctness.

---

### The Real Cost of Irreversibility

Type 1 decisions deserve the caution they receive, but it's worth being precise about what makes a decision irreversible in software.

**External coupling** is the primary source of irreversibility. A decision that external systems depend on is hard to change because you don't control those systems. A public API contract, a data format consumed by partners, an event schema subscribed to by downstream services — these accrue external dependents that each have their own change cycles. The more external coupling a decision creates, the more irreversible it becomes.

**Data is harder to change than code.** A poor abstraction in the code can be refactored. A poor data model that has been running in production for two years, accumulating records in a schema that was wrong from the start, is much harder to address. Data migrations at scale are risky, slow, and expensive. Database schema decisions have a higher effective irreversibility than most code decisions.

**Operational investment creates lock-in.** A team that has built deep expertise in a technology, invested in tooling, and built significant infrastructure around a platform has sunk costs that make switching expensive even if the decision was wrong. This isn't purely a technical concern — it's an organisational one.

**Architectural decisions that become load-bearing.** A decision that initially seemed isolated can become structural as other decisions are built on top of it. Choosing a synchronous request-response model seems straightforward early on; once the entire system assumes synchronous communication and dozens of services are built around it, moving to an event-driven model is an architectural transformation, not a configuration change.

---

### Deferring Decisions Well

The goal is not to avoid decisions indefinitely — it's to make them when the information needed to make them well is available. This requires actively preserving optionality while working with what is known.

**Decide the minimum necessary now.** When starting a new system, the question is not "what is the right architecture" but "what is the minimum architectural commitment needed to build the first thing we need to build." The rest can be decided as the system and the understanding of its requirements evolve.

**Design for replaceability.** Clean interfaces between components make components replaceable. A repository interface that abstracts the database means the database can be changed without touching the use cases. A payment gateway abstracted behind an interface means the payment provider can be switched without rewriting business logic. These abstractions don't prevent the decision — they make it reversible. The cost of the abstraction is paid once; the optionality it preserves can be exercised many times.

**Treat early implementations as spikes.** When building in an area of genuine uncertainty, the first implementation is a learning exercise as much as a delivery. Building it with the explicit understanding that it may be thrown away — or significantly revised — changes how it is approached. Spending three weeks gold-plating an implementation in an area where the requirements aren't yet understood is different from spending three weeks building something that answers specific questions and can be refined once those questions have answers.

**Write down what you don't know.** Architecture decisions often get documented as confident conclusions. The uncertainty that surrounded them — the options considered and rejected, the assumptions made, the questions that remained unanswered — disappears. An architecture decision record that includes the context, the options considered, the decision, and crucially the assumptions on which it rests is far more useful than one that presents only the conclusion. When an assumption turns out to be wrong, the record tells you what needs to be revisited.

---

### The Cost of Deciding Too Early

Early decisions carry a specific kind of cost that compounds over time: they shape all subsequent decisions, which means errors propagate.

A database schema decided before the domain model was properly understood forces the domain model to conform to the schema rather than to the domain. The data model should follow the domain model; when the order is reversed, the domain model is constrained by a representation that was designed before the domain was properly understood. Every subsequent feature is built on a foundation that was wrong before the first line of business logic was written.

An architectural pattern adopted before the system's characteristics are known — its scale, its consistency requirements, its failure modes — might be solving a problem the system doesn't have while creating problems it now does. A complex distributed architecture chosen speculatively for a system that turns out to have modest throughput requirements adds permanent operational complexity with no commensurate benefit.

The language of investment is useful here. An early decision is a bet. The earlier you make it, the less information you have, the more likely the bet is wrong, and the longer you carry the consequences. Deferring the bet doesn't eliminate it — you still have to decide eventually — but it increases the quality of the information available when you do, which increases the probability that the bet is right.

---

### Living With Uncertainty Without Paralysis

The failure mode in the other direction is using uncertainty as a reason to avoid deciding anything. Analysis paralysis is real and expensive. A team that defers every decision because it might be wrong makes no progress and delivers no value.

The practical resolution is to distinguish clearly between the decision and the investigation:

**Timebox the investigation.** Uncertainty is addressed by gathering information, not by waiting. A spike to evaluate two database options, a prototype to test an architectural approach, a conversation with the domain expert to clarify a business rule — these are time-bounded activities that convert uncertainty into information. Running them with a clear question and a fixed timebox produces an answer; leaving them open-ended produces delay.

**Accept reversibility as a substitute for certainty.** For Type 2 decisions, the right question is not "is this definitely correct" but "can we correct it cheaply if it's wrong." If the answer is yes, decide now, move forward, and correct as needed. The cost of a wrong reversible decision is low; the cost of deferring it is the delay it imposes on everything that follows.

**Separate the decision from the implementation.** Sometimes the decision can be made — the approach agreed, the direction set — without committing to the full implementation. Agreeing on an event-driven approach to inter-service communication is a decision; implementing the full messaging infrastructure is a separate step that can follow once the first services are working and the decision has been validated.

**Name the assumptions explicitly.** Making a decision under uncertainty is more defensible when the uncertainty is named. "We're choosing PostgreSQL on the assumption that our data access patterns remain relational; if we need graph traversal at scale we'll revisit this" is a decision with a clear trigger for re-evaluation. It moves forward without pretending to certainty that doesn't exist.

---

### Reversibility as an Architectural Quality

The most resilient architectures are those that treat reversibility as a first-class quality — not just as a nice-to-have but as a design goal.

This shows up in several ways. Interfaces over concrete dependencies. Small, well-bounded services that can be replaced without affecting neighbours. Data models designed for evolution — additive changes, versioned schemas, tolerant readers. APIs that are designed to be extended without breaking existing consumers. Event schemas designed with forward and backward compatibility in mind.

These properties don't make systems immune to wrong decisions. They make wrong decisions cheaper to correct. A system designed for reversibility turns Type 1 decisions into Type 2 decisions — not because the decisions become less significant, but because the architectural properties mean that changing them doesn't require dismantling everything that was built on top of them.

This is one of the deepest values of Clean Architecture as a practice. The dependency rule — pointing dependencies inward toward the domain — means that the outer layers, where the most volatile decisions live (frameworks, databases, external services), are replaceable without touching the inner layers where the most stable and valuable logic lives. The architecture is structured to make the most likely changes the cheapest changes.

---

### When to Decide, When to Defer, When to Explore

A practical heuristic for any significant decision:

**Decide now if:** the decision is blocking progress on something that needs to be built, the information needed to decide well is already available, or the cost of being wrong is low enough that correcting it is straightforward.

**Defer if:** the decision would benefit significantly from information that will become available through building, the cost of being wrong is high and the cost of deferring is low, or the system can be structured so that the decision doesn't need to be made yet without incurring technical debt.

**Explore if:** the uncertainty is genuine and material, neither deciding nor deferring is clearly right, and a bounded investigation would substantially improve the quality of the decision.

The meta-skill is recognising which situation you're in and responding appropriately — rather than defaulting to either premature commitment or indefinite deferral. That recognition, combined with the discipline of preserving optionality where it costs little and committing confidently where the information is good, is what good architectural judgment looks like in practice.
