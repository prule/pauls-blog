---
title: "A Game Plan for Product and Software Development"
layout:     page
draft: false
---


## A Game Plan for Product and Software Development

### The Premise

Most software problems are not technical. A team that consistently delivers working software, adapts to changing requirements, and maintains the capacity to keep improving over time is not doing so because they chose the right framework or the right database. They are doing so because they have internalised a coherent set of practices that compound — each one making the others more effective, and all of them together producing a system that remains comprehensible, changeable, and reliable over its lifetime.

What follows is not a methodology with ceremonies and artefacts. It is a set of principles and practices, grounded in everything covered in this series, that form a coherent approach to building software products that last.

---

### Start With the Domain

Before any code is written, the most valuable investment is understanding the problem deeply enough to model it honestly.

This means working with domain experts — the people who understand the business — to develop a shared vocabulary. Not a glossary created by developers to describe their implementation, but a living language that domain experts recognise as their own. The concepts, rules, and relationships that matter in the business should be visible in the code. When they are, the code becomes a conversation between the business and the engineering team rather than a black box that one translates into the other.

The domain model is the heart of the system. It is the thing that changes when the business changes, and the thing that must remain comprehensible as it does. Investing in a rich domain model — one that encodes business rules explicitly, protects its own invariants, and speaks the language of the domain — is the highest-return technical investment available at the start of a product.

Value objects make domain concepts first-class types, preventing entire categories of bug at compile time. Entities and aggregates protect their own invariants, so the rules that must always be true are enforced at the point of change rather than scattered across validation logic. Domain events make the significant things that happen in the system explicit and auditable.

The domain model is not the database schema. It is not the API contract. It is not the framework's entity model. It is the conceptual heart of the system, expressed in code, independent of any delivery mechanism.

---

### Protect the Domain With Architecture

A domain model is only as good as the boundary that protects it. Without architectural discipline, the domain model gradually accumulates framework dependencies, database concerns, and HTTP concepts — and ceases to be a pure expression of the business.

Clean Architecture provides that boundary. The dependency rule — all dependencies pointing inward, toward the domain — ensures that the business logic is never coupled to the details of how it is delivered or stored. The domain doesn't know about Spring, PostgreSQL, Kafka, or REST. It knows about orders, customers, payments, and shipments. The outer layers — adapters, infrastructure, frameworks — translate between the domain and the outside world.

This separation has practical consequences that compound over time. The domain can be tested without spinning up infrastructure. The database can be changed without touching business logic. A new delivery mechanism — a CLI, a background job, a GraphQL API alongside the REST API — can be added by writing a new adapter, not by rewriting the core. The architecture makes the most common changes the cheapest changes.

The same separation applies at the team level. When the architecture reflects the domain — packages named after business concepts rather than technical layers — new developers can navigate by business knowledge rather than by framework conventions. The codebase screams what it does, not how it was built.

---

### Build Confidence With TDD

The domain model and architecture provide the structure. Test-driven development provides the confidence to keep working in that structure as it evolves.

Writing tests first is not primarily about test coverage. It is about design. A test written before its implementation forces the developer to define the interface before the internals — to experience the API as a consumer before building it. This consistently produces cleaner interfaces, more focused responsibilities, and better-named abstractions than implementation-first development.

The test suite that results is a precise, executable specification of what the system does. It documents behaviour in a form that cannot drift from the code — if the code changes without the test changing, the test fails. It makes refactoring safe — the internals can be restructured aggressively as long as the behaviour expressed in the tests remains correct.

In practice, TDD applied to the domain and use case layers — the innermost rings of the architecture — produces a fast, reliable, infrastructure-free test suite that can be run in seconds and trusted completely. This is the foundation that makes continuous integration viable.

---

### Integrate Continuously

Confidence at the test level enables confidence at the integration level. A fast, trusted test suite makes integrating to mainline multiple times a day cheap rather than anxious.

Continuous integration — real CI, where every developer integrates to a single mainline at least daily — changes the economics of software development. Integration cost is superlinear in batch size: integrating one day of work is cheap, integrating two weeks of work is expensive. CI keeps the batch size small, which keeps the cost of integration low, which keeps the feedback loop tight.

The pipeline that verifies each integration — compile, unit tests, integration tests, static analysis, security scanning — is the team's safety net. It needs to be fast enough that feedback arrives before context is lost, and trusted enough that a green build means something. Flaky tests erode both properties; they are broken windows that must be fixed immediately.

Pull request review is part of the integration cycle, not separate from it. PRs that are too large, or that wait too long for review, recreate the batch size problem in a different form. Small, focused PRs that move through review quickly keep the batch size small at every level of the delivery pipeline.

Clean Architecture makes small PRs structurally natural. Each layer of a feature — the domain model, the use case, the repository implementation, the controller — is a coherent, independently mergeable unit. The architecture provides the seams; the developer exploits them.

---

### Decompose Relentlessly

The ability to deliver in small, frequent increments depends on the ability to decompose work into small, independently valuable pieces. This is a skill that operates at every level — from breaking down an epic into stories, to breaking a story into tasks, to breaking a function into well-named helpers.

The key discipline is vertical slicing. Rather than building horizontal layers — all the database work, then all the service work, then all the UI work — deliver thin end-to-end slices that each represent a coherent piece of behaviour. The first slice handles the simplest case. Subsequent slices add breadth and depth. At every point, the system is in a working state and the progress is demonstrable.

Vertical slicing requires understanding the full problem before starting, which requires investment in domain understanding upfront. The investment pays returns immediately — in earlier feedback, in smaller PRs, in a system that is always in a deployable state rather than perpetually almost-done.

---

### Release Deliberately

Continuous integration makes the codebase always releasable. Feature flags make releasing a deliberate business decision rather than a consequence of deployment.

The separation of deployment from release is one of the most operationally valuable practices available. Code is deployed continuously and safely — each deployment is small, the CI pipeline has verified it, and rollback is a flag toggle rather than a redeployment. Features are released when the business is ready — when marketing has prepared, when the rollout can be gradual, when the right customers are in the right cohort.

This gives the product team control over the user experience without requiring the engineering team to coordinate deployments around launch plans. It gives the engineering team freedom to deploy continuously without waiting for release windows. Both teams operate on their own cadence, connected by the flag that mediates between them.

Trunk-based development is the branching strategy that this approach implies. Long-lived feature branches reintroduce the batch size problem at the branching level. Short-lived branches — merged to mainline within hours or a day — keep the integration continuous and the mainline always current.

---

### Keep Dependencies Current

A product built on a foundation of outdated dependencies is a product accumulating security exposure and upgrade debt simultaneously. Renovate makes dependency updates routine rather than occasional.

The principle is the same as CI applied to dependencies: small, frequent updates are cheaper and safer than large, infrequent ones. A patch update arriving as a PR the day it is released, verified by the CI pipeline, and merged automatically is a non-event. The same patch discovered six months later, requiring navigation through several intermediate versions, is a project.

Keeping dependencies current is a security practice as much as a maintenance practice. The window between a vulnerability being disclosed and a patch being available is narrow; the window between a patch being available and an organisation applying it should be narrower. Automation closes that window by default.

---

### Manage Complexity Actively

Every system accumulates complexity over time. Some of that complexity is essential — the genuine difficulty of the problem being solved. Some of it is accidental — complexity introduced by the solution rather than the problem, which can and should be removed.

The practices in this game plan collectively keep accidental complexity in check. TDD pushes toward small, focused units. Clean Architecture keeps concerns separated. Rich domain models make essential complexity explicit and well-named. Value objects eliminate primitive obsession. Decomposition keeps units at a size where they can be understood as single conceptual objects.

But discipline alone is not enough. Technical debt accumulates despite good practices, and it needs to be managed actively. The approach that works is continuous rather than periodic — the boy scout rule applied consistently produces a codebase that improves incrementally with each delivery, rather than one that degrades until a dedicated cleanup effort is required. Debt taken on deliberately — under genuine time pressure, for a specific known trade-off — should be named, recorded, and repaid with a clear trigger.

Cognitive load is the measure that matters. If working in the codebase requires holding too much in working memory simultaneously, something is wrong — either with the structure of the code, or with the abstractions being used, or with the domain model's fidelity to the domain. Reducing the cognitive load of the codebase is reducing the cost of every future change.

---

### Observe What You Build

A system deployed to production is a system operating in an environment that is fundamentally different from the test environment. Users behave in unexpected ways. Dependencies fail in unexpected ways. Load is distributed unexpectedly. The only way to understand what the system is actually doing is to observe it.

Observability is a design concern, not an operational afterthought. The three signals — logs, metrics, and traces — need to be designed into the system from the start, not added after the first production incident makes their absence painful. Correlation IDs that propagate through every operation, structured logs that can be queried by any field, metrics aligned with the things the team will need to alert on — these are engineering decisions made at coding time that determine the team's operational capability indefinitely.

A system that is observable is a system that can be understood and improved based on real production behaviour. A system that is not observable is one that must be inferred from user reports and instinct.

---

### Document the Why, Not the What

The code documents what the system does. Documentation should cover what the code cannot say: why decisions were made, what alternatives were rejected, what constraints shaped the design, what assumptions underpin it.

Architecture decision records capture the reasoning behind significant architectural choices at the moment when that reasoning is freshest and most complete. Operational runbooks capture the procedures that matter during incidents. Comments in the code explain the non-obvious — the constraint being worked around, the regulatory requirement being met, the edge case being handled deliberately.

Everything else — documentation that restates what the code says, process documentation that nobody reads, speculative documentation for features that don't exist — is maintenance cost without return. Write less, more carefully, as close to the code as possible.

---

### Grow the Team's Capability

The practices in this game plan are not individual skills — they are team skills. A single developer who understands clean architecture, TDD, and domain modelling can apply these principles to their own code. A team that shares these principles produces a codebase with consistent structure, shared vocabulary, and collective ownership.

Growing that shared understanding is one of the most valuable investments a team can make. Code review is a teaching tool when it asks questions rather than issuing corrections, when it distinguishes between things that matter and personal preference, and when it transfers the tacit knowledge that senior developers have accumulated. Pairing and collaborative work — not necessarily strict pair programming, but deliberate working together on hard problems — spreads understanding faster than documentation.

The developer who can decompose a problem well, model a domain faithfully, write code that communicates, and deliver in small frequent increments is more valuable than one with deep knowledge of any particular technology. These are the skills that remain relevant as technologies change, and they are the skills that determine the long-term trajectory of a codebase.

---

### The Compounding Effect

The reason these practices form a coherent game plan rather than a collection of independent techniques is that they compound. CI is more valuable when PRs are small, and PRs are small when the architecture has natural seams, and architecture has natural seams when the domain is modelled well, and the domain model is trustworthy when it is covered by tests written test-first. Observability is more useful when the domain events are explicit. Documentation is more maintainable when it is close to the code. Cognitive load is lower when naming is precise, units are bounded, and primitives have been replaced with value objects.

Each practice reinforces the others. A team that adopts them together does not experience the sum of their individual benefits — it experiences something larger, because the practices create a self-reinforcing cycle: comprehensible code is easier to test, tested code is easier to refactor, refactored code is more comprehensible.

The teams that build the best software over the long term are not the teams with the most talented individuals. They are the teams with the most coherent and consistently applied practices — teams where the code reflects the domain, the architecture protects the domain, the tests specify the behaviour, the pipeline verifies the integration, and the people share enough common understanding to build on each other's work with confidence.

That is the game plan. Not a set of rules to follow, but a way of thinking about software that, applied consistently, produces systems that last.
