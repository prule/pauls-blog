# CLAUDE.md

Guide for AI working in this repository. Two parts: engineering principles distilled from `content/page/software/`, and a writing guide for blog posts on those topics.

The principles are not abstract preferences. They reflect a coherent position on how sustainable software gets built: through compounding practices that manage human cognition, model the domain honestly, and shorten feedback loops. Follow them when writing code, suggesting changes, reviewing PRs, or authoring posts — and when in doubt, read the source article.

---

## Part 1 — Engineering principles

### Governing idea

Software teams that consistently deliver valuable, maintainable software do so through a coherent set of compounding practices — not through individual talent or technology choices. Every recommendation below serves one of three supporting arguments: structure determines speed, small and frequent beats large and infrequent, and practices compound.

Source: `content/page/software/software-engineering-excellence.md` (the pyramid-principle synthesis of the entire collection).

### Cognitive load is the primary constraint

Treat cognitive load as the unifying concern beneath every other practice. Working memory holds 4–7 items; comprehension degrades past that and bugs follow.

- **Why:** Code is read many times more than it is written. The reader must hold the relevant parts of the system in their head to change it safely.
- **How to apply:** Prefer flat over deeply nested. Prefer consistent over clever. Prefer small focused units over large multi-purpose ones. When suggesting a change, ask "does this raise or lower the cognitive load of working in this area?"
- Distinguish *intrinsic* (problem complexity), *extraneous* (solution friction — minimise this), and *germane* (durable mental models — invest in this).
- Source: `cognitive-load.md`.

### Manage essential vs accidental complexity

Brooks' distinction is load-bearing. Essential complexity comes from the domain and must be respected. Accidental complexity comes from the solution and must be relentlessly removed.

- **Why:** Every unnecessary concept, layer, or inconsistency is a tax paid by every future reader.
- **How to apply:** Before adding an abstraction, layer, or pattern, ask what would be lost if it were removed. Delete dead code. Rename misleading names. Flatten unjustified indirection. Prefer simple solutions to clever ones.
- Source: `complexity.md`.

### The domain is the truth

The code should speak the business's language, not the framework's. Model the domain richly; let the architecture reflect what the system *does*, not how it was built.

- **Why:** When code reflects the domain, the gap between business intent and implementation collapses. The code becomes self-documenting and domain experts can recognise their own concepts in it.
- **How to apply:** Use the ubiquitous language consistently. Put business logic *inside* domain objects (rich models), not in surrounding services (anaemic models). Protect invariants with private constructors, factory methods, sealed types, and encapsulated collections. Make illegal states unrepresentable. Package structure should "scream" the domain (`orders/`, `billing/`, `inventory/`) — not the framework (`controllers/`, `services/`, `repositories/`).
- Source: `domain-modelling.md`, `manifesto.md`.

### Value objects over primitives

Wrap domain concepts in their own types. The compiler becomes a domain expert.

- **Why:** Primitives carry no domain meaning. `String`, `Int`, `BigDecimal` allow argument-ordering bugs, cross-currency arithmetic, unit confusion, and silent invalid values. Value objects make these compile errors, not runtime surprises.
- **How to apply:** Identifiers (`OrderId`, `CustomerId`) get distinct types. Money carries currency. Numeric values carry units. Validation lives once, in the constructor. In Kotlin, use `@JvmInline value class` for zero-overhead single-value wrappers.
- Source: `kotlin-value-objects.md`.

### Clean Architecture: dependencies point inward

Business logic at the centre; frameworks, databases, and UIs are details on the outside.

- **Why:** Outer layers (volatile decisions about delivery and storage) should depend on the inner layers (stable business rules). This makes the most likely changes the cheapest changes, keeps the domain testable without infrastructure, and turns layer boundaries into natural PR boundaries.
- **How to apply:** Domain objects import no frameworks. Use cases depend on repository *interfaces*, not implementations. Controllers are humble — translate and delegate. Each boundary translates data into the receiving layer's vocabulary (commands, domain models, results, view models). Don't reach across layers.
- When *not* to apply: thin CRUD with no real business rules. Clean Architecture earns its complexity when the domain justifies it.
- Source: `clean-architecture.md`.

### TDD drives design, not just verification

Red-green-refactor in tight cycles. Write the failing test first; write the simplest code to pass; clean up.

- **Why:** Writing the test first forces you to define the interface as a consumer before the implementation. Painful tests signal painful design. The resulting suite is fast, isolated, and trusted — which is what makes CI viable.
- **How to apply:** Test behaviour, not implementation. Name tests as specifications (`when an order is placed with no items, the order is rejected`). One behaviour per test. Arrange/Act/Assert with blank lines between. Coverage is a side effect, not a goal.
- Source: `tdd.md`.

### Decomposition is the master skill

Operate it at every level: epics into stories, features into vertical slices, classes into single responsibilities, functions into single purposes, problems into failing tests.

- **Why:** Independence is what enables parallel work, small PRs, fast review, and confident refactoring. Tight coupling at any level kills flow at every level.
- **How to apply:** Decompose the *problem*, not the solution (avoid "frontend task / backend task / database task" — those don't deliver value independently). Slice vertically — thin end-to-end behaviour — not horizontally by layer. A function name with "and" in it is a seam. A test that's hard to write is feedback that the unit is wrong.
- Source: `breaking-down-problems.md`.

### Continuous integration: small and frequent

Real CI means daily integration to a single mainline, verified by a fast pipeline. Not "we have a build server."

- **Why:** Integration cost is superlinear in batch size. Small frequent merges are cheap; large infrequent merges are expensive. The economics compound in either direction.
- **How to apply:** Trunk-based development. Branches measured in hours, not days. Pipeline under 10 minutes (5 is better). Broken build is an emergency, not a backlog item. Flaky tests are broken windows — fix or delete. Small PRs (Clean Architecture's layer boundaries make this structurally natural). Review SLAs are explicit, not aspirational.
- Source: `continuous-integration.md`.

### Feature flags separate deployment from release

Deploy continuously and dormantly; release deliberately.

- **Why:** Resolves the tension between CI (integrate constantly) and CD (mainline always releasable) for features that take longer than a day. Deployment becomes a non-event; release becomes a business decision.
- **How to apply:** Distinguish release flags (short-lived, removed after rollout), experiment flags (A/B, also temporary), ops flags (kill switches, may be long-lived), and permission flags (which belong in an entitlements system, not a flag system). Cleanup is part of the lifecycle — flag debt is real debt.
- For larger structural changes, use *branch by abstraction* — introduce an interface, swap implementations, then remove the old one.
- Source: `feature-flags.md`.

### Keep dependencies current with automation

Renovate (or equivalent) makes dependency updates a continuous, low-risk routine.

- **Why:** Same small-and-frequent economics as CI. Patch updates that arrive as individual PRs are cheap; multi-version upgrades forced by a CVE are projects. Staying current is a security practice as much as maintenance.
- **How to apply:** Patch updates auto-merge on green. Minor and major get human review. Group related packages to reduce noise. The merge habit matters as much as the tool — PRs that accumulate without merging defeat the purpose.
- Source: `updating-dependencies.md`.

### Decision-making: distinguish reversible from irreversible

Bezos' Type 1 / Type 2 distinction applies directly to architecture.

- **Why:** Treating reversible decisions with the caution appropriate to irreversible ones is slow without being safer. Treating irreversible decisions casually is reckless.
- **How to apply:** Type 1 (irreversible — public API contracts, data models, fundamental architecture) deserves serious investigation. Type 2 (reversible — internal class names, library choices behind interfaces) should be made quickly and corrected if wrong. Defer commitments where the cost of waiting is low and the information will improve. Design for replaceability — clean interfaces turn Type 1 decisions into Type 2 decisions. Name assumptions explicitly so the trigger for revisiting is clear.
- Source: `decision-making.md`.

### Technical debt: be honest about it

Distinguish deliberate prudent debt (Cunningham's original meaning), deliberate reckless debt, inadvertent debt, and bit rot. Each warrants a different response.

- **Why:** Treating all debt as one category leads to bad prioritisation. Interest rates vary — debt in central, frequently-changed code is expensive; debt in stable peripheral code costs almost nothing.
- **How to apply:** Address debt when you're in the area anyway (boy scout rule). Prioritise where centrality × change-frequency is high. Talk about debt in delivery-cost terms (slower velocity, higher bug rate, riskier changes) — not in technical terms — so it becomes a business conversation. Under time pressure, take debt deliberately in the *outer* layers, protect the domain, never break the build, and clean up immediately after the deadline while context is fresh.
- Source: `tech-debt.md`.

### Observability is a design concern

Logs, metrics, traces — designed in from the start, not bolted on after the first incident.

- **Why:** A system that is opaque in production is a system that cannot be operated, debugged, or improved on evidence.
- **How to apply:** Correlation IDs from day one (very expensive to retrofit). Structured logging, not string concatenation. Log at boundaries, not everywhere. Metrics aligned with what you'll alert on (the four golden signals — latency, traffic, errors, saturation). Instrumentation lives in adapter/infrastructure layers via the decorator pattern; the domain model emits domain events and remains pure. Distinguish liveness ("restart me") from readiness ("don't route to me").
- Source: `observability.md`.

### Documentation: write the why, not the what

Code documents what; documentation documents why, what was rejected, and what assumptions hold.

- **Why:** Code that looks wrong but isn't will be "fixed" by the next developer unless the constraint is recorded. Architecture decision records (ADRs) capture context at the moment it's freshest.
- **How to apply:** ADRs in `/docs/decisions`, versioned with code. Comments explain non-obvious decisions, not what the code already says. Operational runbooks for procedures used under pressure. Tests as specifications. Generate API docs from code annotations. Treat outdated documentation as a bug.
- Source: `tech-docs.md`.

### Write code for the reader

The compiler isn't your audience.

- **Why:** Code is written once and read many times. The hundreds of micro-decisions during writing — naming, extraction, blank lines, level of abstraction — determine whether the next reader understands immediately or struggles for twenty minutes.
- **How to apply:** One level of abstraction per function. Blank lines between logical sections (paragraph breaks). Brevity is not a virtue if it costs comprehension — named intermediate variables beat dense one-liners. Fail fast and clearly at the right boundary, with messages that say what was wrong and what was expected. No commented-out code, no stale TODOs, no half-finished refactors.
- Source: `readable-code.md`, `coding-tips.md`.

### Coding tips applied consistently

The micro-decisions that produce code which is comprehensible by default. Headlines (full detail in `coding-tips.md`):

Control flow: return early, avoid `else` after `return`, prefer positive conditions, exhaust sealed types (no `else` branches that swallow new cases). Naming: booleans as questions (`isVerified`, `hasDiscount`), name what something *represents* not its type, include units in numeric names. Functions: do one thing at one level, keep the happy path at the top, limit parameters (>3 is a signal of an unnamed concept), avoid boolean parameters, return rather than mutate. Types: avoid primitive obsession, encode constraints in types, prefer immutability, make null meaningful or eliminate it. Classes: simple constructors, minimum public surface, composition over inheritance, never expose mutable internal collections. Errors: fail fast, exceptions for exceptional conditions, no silent swallowing, catch specific not broad. Tests: name as specs, one behaviour each, AAA with blank lines, test behaviour not implementation. General: leave code better than you found it, delete unused code, extract magic numbers, treat warnings as errors, separate cleanup from feature changes, read your own diff before opening the PR.

When in doubt, **make it boring**. Interesting code is interesting to write; boring code is valuable to maintain.

### Reading unfamiliar code

A distinct skill, used constantly, rarely taught.

- **How to apply:** Start with the *why*, not the *what*. Read tests before implementation — they're the specification. Find entry points and trace one request through. Map structure before reading detail. Use the version history to recover lost context. Build a glossary. Form hypotheses and test them; revise the model as evidence demands. Make a small change to convert passive understanding into active knowledge.
- Source: `reading-code.md`.

### Configurable systems and the rule of three

Don't abstract too early; don't refuse to abstract when the pattern is genuinely there.

- **Why:** A premature framework fights every use case. A missed framework duplicates structure across instances and lets inconsistency accumulate.
- **How to apply:** First time, just solve it. Second time, solve it again and notice the duplication. Third time, extract the abstraction. The right abstraction has a name from the domain (star schema, pipeline, workflow), is stable, compresses without losing information, and breaks honestly when the pattern doesn't fit (escape hatches like `customQueryOverride` are a signal, not a feature).
- Source: `configurable-systems.md`.

### Collaboration as a spectrum

Strict pair programming is one point on a spectrum, not the only valid mode.

- **How to apply:** Match the mode to the problem — frame together and execute separately for well-understood work; pair side-by-side on novel problems and tricky bugs; async-with-touchpoints respects deep work; rubber-ducking-with-teeth (questions, not answers) often beats demonstrating. In mentoring, stay in questioning mode longer than feels comfortable; narrate intuitive decisions out loud (tacit knowledge doesn't transfer otherwise); let ownership transfer gradually.
- Source: `pairing.md`.

### REST and HATEOAS

Most "REST APIs" sit at Richardson level 2. Level 3 (hypermedia) is rare but has real practical value.

- **Why:** Server-driven affordances mean the link set on a response is a capability manifest — the client renders actions based on what the server says is currently possible, rather than reimplementing permission and state-transition logic that drifts. Same endpoint, different links for different roles or states.
- **How to apply:** When designing internal APIs where client/server drift is a real problem, consider HAL/JSON:API for `_links`. Don't introduce HATEOAS for simple CRUD with a single tightly-coupled client.
- Source: `rest-hateoas.md`.

### Team structure mirrors domain structure

Conway's Law isn't a suggestion. Team Topologies provides the practical model.

- **How to apply:** Stream-aligned teams (5–8 people) own a bounded context end-to-end — product, design, dev, test, operate. Platform teams build internal products that reduce stream-aligned teams' cognitive load (and are judged on whether teams choose to use them). Enabling teams work alongside stream-aligned teams to build a capability, then leave. Complicated subsystem teams own components needing deep specialism. Choose interaction modes deliberately: collaboration (temporary, for unknowns), X-as-a-Service (stable interface), facilitation (enabling).
- Source: `game-plan.md`, `software-engineering-excellence.md`.

### How the practices compound

No practice in this collection stands alone. TDD makes CI viable. Clean Architecture makes small PRs natural. Rich domain models make use cases thin. Value objects make the type system an ally. Feature flags make trunk-based development safe. Stream-aligned ownership makes observability a priority. Each reinforces the others.

When recommending a practice, prefer suggestions that strengthen the surrounding practices over ones that work in isolation. When critiquing a piece of code or design, look for the place in this network where the leverage is highest — usually the domain model or the architectural boundary.

---

## Part 2 — Writing guide for blog posts

When authoring or editing posts under `content/page/software/`, follow these conventions. They reflect how the existing collection is written.

### Voice and tone

- **Audience:** senior developers and the audiences they sit alongside (tech leads, staff engineers, engineering managers). Assume they know the basics; don't over-explain. Don't talk down.
- **Tone:** measured, considered, technical. Like a senior colleague thinking out loud, not a vendor selling a methodology.
- **No superlatives, no sensationalism.** Avoid "revolutionary," "game-changing," "best-in-class," "incredible." If something is significant, demonstrate it with reasoning or example, don't assert it with adjectives.
- **No hype words for tools or technologies.** Describe what they do and what they cost.
- **Caveats and trade-offs are part of the substance.** Every practice has a cost; name it. The "When *not* to apply" or "The Cost" section is part of the credibility, not a hedge.

### Structure: McKinsey pyramid principle

Every post should be readable top-down: governing idea first, then mutually exclusive supporting arguments, each backed by evidence and reasoning.

- **Lead with the central claim.** A post on TDD opens with what TDD inverts and why. A post on cognitive load opens with the bottleneck and the three forms.
- **Section headings carry the argument.** A reader scanning headings should get the spine of the piece without reading the body. Avoid generic headings like "Background" or "Conclusion."
- **Each section makes one point.** If a section contains two arguments, split it.
- **Conclude by tying back to the governing idea**, often by showing how this practice composes with others in the collection.

### Length and depth

Existing posts run from short (rest-hateoas, configurable-systems — ~150 lines) to long (cognitive-load, complexity, software-engineering-excellence — 200+ lines). Length follows substance, not the other way round. Don't pad; don't truncate something that needs space to be understood.

### Use of examples

- **Code examples are Kotlin** unless the post is about a different language or platform.
- **Show before-and-after pairs** where the contrast is the point ("Mixed levels — reader must process detail and intent simultaneously" / "Consistent level — reader understands intent immediately"). Both versions should be runnable and realistic.
- **Use the running example domains** that already appear in the collection: orders, customers, payments, shipments, money, products, inventory. This keeps the collection coherent and lets readers transfer understanding between posts.
- **Examples illustrate the principle, not exhaust it.** A short example that makes one point lands better than a long example that tries to demonstrate everything.

### Cross-references

Posts in this collection are deliberately interconnected. When a topic in one post relates to another, name the connection ("This is the same dependency inversion principle from Clean Architecture applied to observability"). The compounding nature of the practices is part of the argument, and explicit connections reinforce it.

The two synthesis posts — `manifesto.md` (the spirit) and `software-engineering-excellence.md` (the pyramid principle structure) — are the canonical references for how the collection fits together. New posts should slot into that structure, not contradict it.

### Frontmatter format

All posts use Hugo frontmatter. Match the existing pattern:

```yaml
---
title: "<Title Case>"
layout:     page
draft: false
description: "<One-sentence description, complete sentence with full stop. Used in listings and meta tags.>"
---
```

Posts that should be visible on the software index live in `content/page/software/` and are linked from `_index.md`. The `draft: false` is required for the post to render in production.

### What to avoid

- **Bullet-point soup.** Use prose. Bullets are for short, parallel items where the structure is genuinely a list. The existing collection uses paragraphs heavily and bullets sparingly.
- **Jargon for its own sake.** If a term needs explaining, explain it (Brooks' essential vs accidental complexity is introduced before it's used).
- **Originality theatre.** Don't claim novelty for ideas with established attributions. Cite the source — Brooks, Cunningham, Bezos, Fielding, Beck, Fowler, Skelton & Pais, Evans, Martin — when introducing the concept.
- **Process documentation disguised as engineering writing.** Posts are about how to think, not about how a particular team's sprint cadence works.
- **Closing exhortations.** Don't end with "you should adopt these practices today." End with an observation that ties the practice to the broader pattern.

### Editing existing posts

When updating a post (the collection is explicitly described as living documents):

- Preserve the voice. Match sentence rhythm, paragraph length, and example style.
- Preserve cross-references unless the referenced content has changed.
- Preserve the frontmatter, especially the `description` field — update it only if the post's substance has shifted.
- Don't introduce new running-example domains casually — the consistency of orders/customers/payments/shipments across posts is deliberate.

---

## Part 3 — Working in this repo

- This is a Hugo blog. Software articles live at `content/page/software/<slug>.md`.
- Built output sits under `public/`. Don't edit it directly — it regenerates.
- The index page is `content/page/software/_index.md`.
- Two synthesis posts sit at the top of the conceptual hierarchy: `manifesto.md` (the spirit) and `software-engineering-excellence.md` (the pyramid synthesis). Read these first when getting oriented.
