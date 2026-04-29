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

### Structure the Organisation Around the Domain

How a team is structured is not a management concern separate from software architecture — it is an architectural decision with direct consequences for the software produced. Conway's Law states that organisations design systems that mirror their communication structures. If the team structure is wrong, the architecture will be wrong regardless of the technical skill of the individuals involved.

Team Topologies, by Matthew Skelton and Manuel Pais, provides a practical model for structuring engineering organisations around fast flow of change. It describes four fundamental team types and three interaction modes that, applied thoughtfully, align the organisation with the domain and reduce the friction of delivery.

**Stream-aligned teams** are the primary building block. A stream-aligned team owns a slice of the business domain end-to-end — from understanding user needs through to operating the software in production. It has all the skills needed to deliver independently: product, design, development, testing, and operational capability. It is aligned to a flow of business value rather than to a technical layer or a functional discipline.

The stream-aligned team is the expression of the domain model at the organisational level. Just as the domain model separates concerns into bounded contexts with clear interfaces, the organisation separates concerns into stream-aligned teams with clear ownership. A team that owns the orders domain owns everything about orders — the model, the service, the database, the API, the monitoring. It does not depend on a separate DBA team to change a schema, a separate ops team to deploy, or a separate QA team to test. Those dependencies are the friction that slows delivery; eliminating them is the point.

A well-functioning stream-aligned team is small — typically five to eight people, consistent with what Skelton and Pais describe and with Amazon's well-known two-pizza rule. Small teams have low coordination overhead, high shared context, and clear ownership. Every member knows what the team is building and why. Decision-making is fast because it doesn't require escalation or cross-team negotiation for routine choices.

**Platform teams** exist to reduce the cognitive load of stream-aligned teams. Rather than every stream-aligned team building and maintaining its own deployment pipeline, observability stack, and infrastructure tooling, the platform team builds and operates these as internal products. The platform team's customers are the stream-aligned teams; their measure of success is reducing the time those teams spend on undifferentiated infrastructure work.

A platform team that imposes mandatory processes, requires tickets for routine operations, or produces tooling that is harder to use than the alternatives is failing its purpose. The platform should feel like a product — with good documentation, self-service capability, and responsiveness to the needs of its users. The test is whether stream-aligned teams choose to use the platform because it genuinely accelerates them, not because they are required to.

**Enabling teams** are temporary by design. When stream-aligned teams need to acquire a capability they don't have — implementing observability practices, adopting event-driven architecture, improving test coverage — an enabling team works alongside them for a bounded period to build that capability. The enabling team's goal is to make itself unnecessary: once the capability is embedded in the stream-aligned team, the enabling engagement ends. This is the organisational expression of the mentoring principle — the best teaching makes itself gradually unnecessary.

**Complicated subsystem teams** own components that require deep specialist knowledge — a machine learning model, a complex financial calculation engine, a high-performance data processing pipeline. These components are too complex for stream-aligned teams to own alongside their other responsibilities, but they serve multiple stream-aligned teams. The complicated subsystem team owns the component; stream-aligned teams consume it through a well-defined interface.

---

### Align Team Boundaries With Domain Boundaries

The relationship between domain modelling and team structure is not incidental. Bounded contexts — the natural divisions of the domain, each with its own model and its own language — are the right basis for team boundaries.

A bounded context that is owned by a single stream-aligned team has a clear owner, a clear purpose, and a team whose shared vocabulary matches the domain vocabulary. The team's conversations, the code, and the domain model all use the same language. Changes to the domain are made by the team that understands it. The knowledge needed to make good decisions lives with the people making them.

When team boundaries cut across domain boundaries — when a single bounded context is owned by multiple teams, or when a single team owns multiple unrelated domains — the result is the coordination overhead and the architectural erosion that Conway's Law predicts. Two teams that jointly own a domain must coordinate every change. The shared model gradually develops inconsistencies as each team pulls it toward their own concerns.

Getting the boundaries right is hard and rarely perfect at the outset. Domains are discovered as much as designed — the right bounded context boundaries often only become clear through building. The practical approach is to start with boundaries that reflect the current understanding of the domain, hold them loosely, and be willing to redraw them as understanding deepens. Splitting a team that has grown too large, merging two teams whose domains have converged, or extracting a complicated subsystem team from a stream-aligned team that has acquired deep specialist knowledge — these are normal adjustments in a system designed for evolution.

---

### Design Interactions Between Teams

Team Topologies describes three interaction modes that govern how teams work with each other. Choosing the right mode for each relationship reduces friction and keeps dependencies explicit.

**Collaboration** is appropriate when two teams are working on a problem that neither can solve independently — exploring a new domain together, establishing an interface between two systems, or building shared understanding in an area of genuine uncertainty. Collaboration is intensive and productive but expensive in coordination overhead. It should be temporary — once the problem is solved and the interface is established, the teams should move to a different interaction mode.

**X-as-a-Service** is the mode for a well-defined, stable interface. One team provides a capability; another team consumes it. The consuming team does not need to understand the internals; the providing team does not need to coordinate with the consuming team for routine changes. This is the relationship between a platform team and a stream-aligned team, between a complicated subsystem team and its consumers, and between any two teams whose interface is well understood. It is the organisational equivalent of a well-designed API.

**Facilitation** is the mode for enabling teams. The enabling team works alongside the stream-aligned team, not for them. It brings knowledge and technique; the stream-aligned team retains ownership and accountability. The facilitation engagement is time-bounded and success is measured by the capability remaining in the stream-aligned team after the enabling team moves on.

Being explicit about which mode applies to each team relationship prevents a common failure: two teams that should be in X-as-a-Service mode accidentally operating in collaboration mode, creating ongoing coordination overhead for what should be a stable interface. Or an enabling team that transitions into a platform team without anyone noticing, gradually taking ownership of things that should belong to stream-aligned teams.

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

Team Topologies uses cognitive load as a first-class measure of team health. A stream-aligned team whose cognitive load is too high — because the domain is too large, the system too complex, or the platform too immature — cannot sustain a fast flow of change. Splitting a domain, investing in platform capability, or running an enabling engagement are all interventions that reduce cognitive load at the team level. The same principle that applies to code applies to teams: comprehensible scope is sustainable scope.

---

### Observe What You Build

A system deployed to production is a system operating in an environment that is fundamentally different from the test environment. Users behave in unexpected ways. Dependencies fail in unexpected ways. Load is distributed unexpectedly. The only way to understand what the system is actually doing is to observe it.

Observability is a design concern, not an operational afterthought. The three signals — logs, metrics, and traces — need to be designed into the system from the start, not added after the first production incident makes their absence painful. Correlation IDs that propagate through every operation, structured logs that can be queried by any field, metrics aligned with the things the team will need to alert on — these are engineering decisions made at coding time that determine the team's operational capability indefinitely.

A system that is observable is a system that can be understood and improved based on real production behaviour. A system that is not observable is one that must be inferred from user reports and instinct.

Stream-aligned teams that own their software in production — rather than handing it off to a separate operations function — have the strongest incentive to build observability in from the start. When the team that builds the software is also the team that gets paged when it breaks, observability stops being an optional extra and becomes an integral part of what it means to ship.

---

### Document the Why, Not the What

The code documents what the system does. Documentation should cover what the code cannot say: why decisions were made, what alternatives were rejected, what constraints shaped the design, what assumptions underpin it.

Architecture decision records capture the reasoning behind significant architectural choices at the moment when that reasoning is freshest and most complete. Operational runbooks capture the procedures that matter during incidents. Comments in the code explain the non-obvious — the constraint being worked around, the regulatory requirement being met, the edge case being handled deliberately.

Everything else — documentation that restates what the code says, process documentation that nobody reads, speculative documentation for features that don't exist — is maintenance cost without return. Write less, more carefully, as close to the code as possible.

---

### Grow the Team's Capability

The practices in this game plan are not individual skills — they are team skills. A single developer who understands clean architecture, TDD, and domain modelling can apply these principles to their own code. A team that shares these principles produces a codebase with consistent structure, shared vocabulary, and collective ownership.

Growing that shared understanding is one of the most valuable investments a team can make. Code review is a teaching tool when it asks questions rather than issuing corrections, when it distinguishes between things that matter and personal preference, and when it transfers the tacit knowledge that senior developers have accumulated. Pairing and collaborative work — not necessarily strict pair programming, but deliberate working together on hard problems — spreads understanding faster than documentation.

The enabling team model formalises this at the organisational level. When a capability needs to be built across multiple stream-aligned teams — adopting a new testing approach, embedding observability practices, introducing domain-driven design — an enabling team that works alongside each stream-aligned team in turn is more effective than training courses, mandates, or documentation alone. Capability built through working together on real problems sticks. Capability delivered as a presentation does not.

The developer who can decompose a problem well, model a domain faithfully, write code that communicates, and deliver in small frequent increments is more valuable than one with deep knowledge of any particular technology. These are the skills that remain relevant as technologies change, and they are the skills that determine the long-term trajectory of a codebase and a team.

---

### The Compounding Effect

The reason these practices form a coherent game plan rather than a collection of independent techniques is that they compound — and this is as true at the organisational level as it is at the code level.

Stream-aligned teams with clear domain ownership make domain-driven design natural — the team and the model share the same boundary. Clean Architecture within those teams makes CI viable — the layer boundaries are the PR boundaries. TDD makes the architecture trustworthy — the behaviour is specified and the refactoring is safe. Feature flags make CI compatible with product release management — deploy continuously, release deliberately. Observability makes production ownership sustainable — the team can see what their software is doing. Cognitive load management keeps teams productive at scale — the scope is comprehensible, the platform handles the undifferentiated work, and the enabling teams build capability that persists.

Each practice reinforces the others. A team that adopts them together does not experience the sum of their individual benefits — it experiences something larger, because the practices create a self-reinforcing cycle. Small teams with clear ownership produce comprehensible codebases. Comprehensible codebases are easier to test. Tested codebases are safer to refactor. Refactored codebases stay comprehensible. The cycle compounds in the right direction.

The teams that build the best software over the long term are not the teams with the most talented individuals. They are the teams with the most coherent and consistently applied practices — teams where the code reflects the domain, the architecture protects the domain, the tests specify the behaviour, the pipeline verifies the integration, the team structure aligns with the domain boundaries, and the people share enough common understanding to build on each other's work with confidence.

That is the game plan. Not a set of rules to follow, but a way of thinking about software — and the organisations that build it — that, applied consistently, produces systems and teams that last.
