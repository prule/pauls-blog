---
title: "A Pyramid Principle Structure - Software Engineering Excellence"
layout:     page
draft: false
description: "This attempts to apply the McKinsey Pyramid Principle to these software development thoughts and practices."
---

> This attempts to apply the McKinsey Pyramid Principle to these software development thoughts and practices.

{{< notice type="info" title="Amazon Affiliates Link" >}}
Buy on Amazon [The Pyramid Principle](https://amzn.to/4uooYee)
{{< /notice >}}


## Software Engineering Excellence — A Pyramid Principle Structure

The McKinsey Pyramid Principle organises thinking from the top down: a single governing idea, supported by a small number of mutually exclusive and collectively exhaustive arguments, each supported by evidence and reasoning. What follows applies that structure to everything covered in this series.

---

### The Governing Idea

**Software teams that consistently deliver valuable, maintainable software do so through a coherent set of compounding practices — not through individual talent or technology choices.**

---

### Three Supporting Arguments

**1. Structure determines speed.**
The way a codebase and an organisation are structured determines how fast the team can move — not just today, but sustainably over time. Poor structure creates friction that compounds. Good structure creates leverage that compounds.

**2. Small and frequent beats large and infrequent.**
At every level — commits, PRs, deployments, team sizes, dependency updates, feedback loops — smaller batches and higher frequency produce better outcomes than larger batches and lower frequency. The economics are consistent and the evidence is overwhelming.

**3. Practices compound.**
No practice in this series stands alone. Each one makes the others more effective. The teams that apply them together experience something larger than the sum of individual benefits.

---

### Argument 1: Structure Determines Speed

#### 1.1 Domain structure drives everything else

The domain model is the heart of the system. All other structural decisions should serve it.

- **Model the domain richly.** Entities, value objects, and aggregates encode business rules explicitly. The domain protects its own invariants. Invalid states are unrepresentable, not just unlikely.
- **Use the ubiquitous language.** The vocabulary of the business should appear in the code. When domain experts and developers share a language, the translation layer between business intent and technical implementation disappears.
- **Value objects over primitives.** Wrapping primitives in domain types makes the compiler a domain expert. Argument-ordering bugs, cross-currency arithmetic, unit confusion — these become compile errors rather than runtime surprises.
- **Make illegal states unrepresentable.** Sealed types, private constructors, and validated value objects push constraints into the type system. Each constraint encoded at compile time is a check that can never be forgotten at runtime.

#### 1.2 Architecture protects the domain

The domain model is only valuable if it remains pure. Architectural discipline enforces that purity.

- **The Dependency Rule.** All dependencies point inward toward the domain. Frameworks, databases, and delivery mechanisms are details. They depend on the domain; the domain does not depend on them.
- **Layers earn their existence.** Each architectural layer should have a clear responsibility and a clear boundary. Layers that don't earn their keep add indirection without value.
- **The architecture should scream the domain.** Package structure reflects business concepts, not framework conventions. A new developer navigates by domain knowledge, not by knowing the framework.
- **Clean Architecture enables small PRs.** Layer boundaries are PR boundaries. A feature arrives as a sequence of small, independently mergeable increments — domain model, use case, repository, controller — each coherent and reviewable in isolation.

#### 1.3 Team structure mirrors domain structure

Conway's Law is not a suggestion. The system will reflect the communication structure of the organisation that builds it. Aligning team structure with domain structure is an architectural decision.

- **Stream-aligned teams own domains end-to-end.** A small team — five to eight people — owns a bounded context from understanding user needs through to operating the software in production. It has all the skills needed to deliver independently.
- **Platform teams reduce cognitive load.** Deployment pipelines, observability infrastructure, and developer tooling built as internal products free stream-aligned teams from undifferentiated infrastructure work. The platform is a product; its users are internal teams.
- **Enabling teams build capability.** When stream-aligned teams need a skill they don't have, an enabling team works alongside them for a bounded period. The goal is to make itself unnecessary. Capability built through working together persists; capability delivered through presentations does not.
- **Team boundaries should match domain boundaries.** A bounded context owned by a single team has a clear owner, a consistent model, and a team whose vocabulary matches the domain. Cross-cutting ownership creates coordination overhead and architectural erosion.
- **Cognitive load is a team health metric.** A team whose scope is too large cannot sustain fast flow. Splitting a domain, investing in platform capability, or running an enabling engagement are interventions that reduce cognitive load at the team level — the same principle that applies to code.

---

### Argument 2: Small and Frequent Beats Large and Infrequent

#### 2.1 Integration should be continuous

Integration cost is superlinear in batch size. The economics of small, frequent integration are unambiguous.

- **Real CI means daily integration to mainline.** Not a pipeline that runs against long-lived feature branches — genuine integration to a single shared mainline at least daily.
- **The pipeline is the team's safety net.** Fast, reliable, comprehensive enough that a green build is meaningful. Under ten minutes. Flaky tests are broken windows — fixed immediately or deleted.
- **Trunk-based development is the implied branching strategy.** Short-lived branches measured in hours, not days. Feature flags handle incomplete features. Long-lived branches reintroduce the batch size problem at the branching level.
- **PR review is part of the integration cycle.** Large PRs that wait days for review recreate the batch size problem in a different form. Small PRs, review agreements, and pairing as a review bypass keep the flow continuous.

#### 2.2 Delivery should be incremental

The ability to deliver incrementally depends on the ability to decompose work into small, independently valuable pieces.

- **Vertical slicing over horizontal layering.** Each increment delivers a thin end-to-end slice of behaviour. The system is always in a working state. Progress is always demonstrable.
- **Decomposition is a core skill.** Finding the natural seams in a problem — the domain boundaries, the behavioural boundaries, the dependency boundaries — is what makes incremental delivery possible. It operates at every level from architecture to function.
- **TDD drives decomposition at the code level.** The red-green-refactor cycle is a decomposition rhythm. Each cycle is a small, coherent increment. A test that is hard to write is a signal that the decomposition is wrong.

#### 2.3 Deployment should be separated from release

Decoupling the technical act of deploying from the business act of releasing removes the coordination overhead that makes deployments expensive.

- **Feature flags separate deployment from release.** Code is deployed continuously and dormantly. Features are released deliberately, when the business is ready, to the right cohort, at the right time.
- **Deployment becomes a non-event.** Small, frequent, pipeline-verified deployments accumulate no risk. Rollback is a flag toggle. The gap between what is in production and what is in the repository is measured in hours.
- **Release becomes a business decision.** Marketing, product, and customer success can coordinate launches independently of deployment schedules. Engineering deploys; the business releases.

#### 2.4 Dependencies should be updated continuously

The same economics that make continuous integration valuable apply to dependencies.

- **Renovate automates the cadence.** Every dependency update arrives as a PR, verified by the CI pipeline, with changelog context. Small, frequent updates are cheap. Large, deferred updates are projects.
- **Staying current is a security practice.** The window between vulnerability disclosure and patch availability is narrow. Automation closes the window between patch availability and adoption.
- **Patch updates auto-merge on green.** A trusted pipeline makes automatic merging of patch updates safe. Minor and major updates receive human review. The calibration is proportionate to risk.

---

### Argument 3: Practices Compound

#### 3.1 Technical practices reinforce each other

Each practice makes the adjacent practices more effective. The compounding is structural, not incidental.

- **TDD makes CI viable.** Tests written test-first are fast, isolated, and reliable — the properties a CI pipeline depends on. The tight cycle also produces small, focused commits naturally.
- **Clean Architecture makes small PRs natural.** Layer boundaries are PR boundaries. The architecture provides the seams that decomposition exploits.
- **Rich domain models make use cases thin.** When the domain enforces its own rules, the use case orchestrates rather than validates. Thin use cases are easier to test, easier to understand, and easier to change.
- **Value objects make the type system an ally.** Each domain concept given its own type is a category of bug that cannot be introduced. The compiler enforces what would otherwise require tests or luck.
- **Feature flags make trunk-based development safe.** Incomplete features are invisible in production. The concern that prevents frequent integration to mainline — half-built features — is addressed structurally.

#### 3.2 Organisational practices reinforce technical practices

The compounding extends from the code to the team to the organisation.

- **Stream-aligned team ownership makes observability a priority.** When the team that builds the software is also the team that operates it in production, observability stops being optional. The incentive and the capability are in the same place.
- **Small teams make domain modelling tractable.** A small team that owns a bounded context shares a vocabulary, builds deep domain knowledge, and produces a model that reflects genuine understanding. A large team fractured across a domain produces inconsistency.
- **Enabling teams make practices spread.** Technical practices embedded through working alongside a stream-aligned team on real problems persist. The enabling model is the organisational expression of collaborative learning.
- **Platform investment multiplies stream-aligned team productivity.** Every hour a stream-aligned team doesn't spend on deployment infrastructure, observability tooling, or security scanning is an hour spent on the domain. The multiplier compounds across every stream-aligned team the platform serves.

#### 3.3 The feedback loops accelerate learning

The practices together create feedback loops at every level that accelerate the team's ability to learn and respond.

- **Tests give immediate feedback on design.** A test that is hard to write is feedback about the design. Received continuously, this feedback shapes the codebase incrementally toward better structure.
- **CI gives immediate feedback on integration.** A pipeline failure is feedback about a breaking change, received within minutes rather than at the next release attempt.
- **Observability gives feedback from production.** Real usage reveals what no test environment can. A team that can observe its software in production can improve it based on evidence rather than assumption.
- **Small deployments give feedback on change.** A deployment that covers one day of work is easy to reason about when something goes wrong. A deployment that covers two weeks of work is not. Small deployments make the relationship between change and effect visible.
- **Code review gives feedback on comprehensibility.** The reviewer experiences the code as a reader, not as the author. Consistent, high-quality review gives the author feedback on whether their code communicates — the feedback that writing for the reader requires.

---

### Managing the Inevitable Entropy

#### Complexity must be actively managed

Every system accumulates complexity. The practices above slow the accumulation; they do not stop it.

- **Essential complexity must be respected.** The genuine difficulty of the domain cannot be removed. It can be modelled faithfully, named well, and expressed clearly — but not eliminated.
- **Accidental complexity must be relentlessly removed.** Poor naming, unnecessary indirection, inconsistency, and accumulated workarounds add cognitive load without adding value. The boy scout rule applied consistently removes them incrementally.
- **Cognitive load is the measure.** If working in the codebase requires holding too much simultaneously, something is wrong. Reducing cognitive load is reducing the cost of every future change.

#### Technical debt must be managed honestly

- **Deliberate debt has a plan.** Debt taken on knowingly, under genuine time pressure, with a clear trigger for repayment, is legitimate. Debt accumulated through inattention is not.
- **Interest rates vary.** Debt in frequently-changed, central code has a high interest rate. Debt in stable, peripheral code has a low one. Address high-rate debt; tolerate low-rate debt.
- **The conversation must be honest.** Debt described in terms of its delivery consequences — slower velocity, higher bug rate, riskier changes — is a business conversation. Debt described in technical terms is ignored.

#### Decisions must be made at the right time

- **Type 1 decisions deserve caution.** Irreversible decisions — public API contracts, data models, fundamental architectural choices — warrant serious investigation before commitment. Getting them wrong is expensive.
- **Type 2 decisions should be made quickly.** Reversible decisions should be made with the information available now and corrected cheaply if wrong. Treating them with Type 1 caution is waste.
- **Defer to preserve optionality.** Decisions made before the information needed to make them well is available produce worse outcomes than decisions deferred until that information exists. Design for replaceability; commit when you must.

---

### The Governing Idea, Restated

Software teams that consistently deliver valuable, maintainable software do so through a coherent set of compounding practices — not through individual talent or technology choices.

Those practices operate at three levels simultaneously. At the code level: domain modelling, value objects, clean architecture, TDD, and writing for the reader. At the delivery level: CI, trunk-based development, feature flags, vertical slicing, and continuous dependency management. At the organisational level: stream-aligned teams, platform investment, enabling capability, and aligning team boundaries with domain boundaries.

Each level reinforces the others. The domain model shapes the architecture shapes the team structure. The team structure shapes the delivery practice shapes the code quality. The code quality shapes the speed of delivery shapes the organisation's capacity to learn and respond.

The teams that get this right don't do so through a single breakthrough insight or a wholesale adoption of a named methodology. They do so through consistent application of coherent principles — each decision, each PR, each team boundary, each architectural choice made in service of the same goal: a system and an organisation that remain comprehensible, changeable, and productive over their lifetime.
