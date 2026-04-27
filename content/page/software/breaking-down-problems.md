---
title: "Breaking Down Problems"
layout:     page
draft: false
---

## Breaking Down Problems — Decomposition as a Core Skill

### Why It's a Skill, Not a Step

Decomposition is often treated as a planning activity — something that happens in a refinement session, produces a list of tickets, and is then considered done. The tickets get handed to developers who implement them. This misses what decomposition actually is.

Decomposition is a continuous cognitive skill exercised throughout development — in planning, yes, but also when sitting down to write a test, when deciding where a class boundary should be, when a piece of code resists being understood. The developer who can break a problem down well works differently at every level: they write smaller functions, cleaner interfaces, more focused tests, and smaller PRs. The skill compounds across every scale of work.

It is also one of the hardest skills to develop, because it requires understanding a problem well enough to see its natural structure — and that understanding often only arrives through the process of working on it.

---

### What Good Decomposition Feels Like

A well-decomposed problem has a particular quality: each piece is coherent on its own terms, the pieces fit together without fighting each other, and the boundaries between them reflect something real about the domain rather than arbitrary lines drawn for convenience.

When decomposition is wrong, you feel it. A subtask that can't be completed without another subtask being done first — but that dependency wasn't visible at planning time. A PR that can't be reviewed because the reviewer needs context from a different PR that isn't merged yet. A function that does two things and can only be tested by testing both simultaneously. A class that needs to know about things that have nothing to do with its stated purpose. These are all signals that the decomposition has a problem.

Good decomposition produces independence. Each piece can be understood, built, tested, and delivered without requiring the others to be present simultaneously. That independence is what enables parallel work, small PRs, fast review, and confident refactoring.

---

### Starting With the Problem, Not the Solution

The most common decomposition failure is jumping to implementation structure before the problem is understood. Breaking a problem into "frontend task, backend task, database task" is decomposing a solution, not a problem. It reflects how the code will be built rather than what the system needs to do. The result is tasks that are technically separable but semantically coupled — the backend task is meaningless without the frontend task, and neither can be validated in isolation.

Starting with the problem means asking: what behaviour needs to exist that doesn't exist now? What decisions need to be made? What is unknown? A problem decomposed by behaviour produces tasks that each represent a coherent capability — something that can be specified, built, tested, and demonstrated independently. A feature to allow customers to cancel their subscription decomposes into: the cancellation policy rules, the state transition in the domain, the confirmation flow in the UI, the downstream effects on billing and access. Each of these has meaning on its own; each can be reasoned about independently.

The shift from solution decomposition to problem decomposition is subtle but significant. It changes what the tasks represent, which changes how they can be worked on and verified.

---

### Finding the Natural Seams

Every problem has natural seams — points where it divides cleanly along boundaries that reflect something real. Finding them is the core of the skill.

**Domain boundaries.** In a well-modelled domain, the concepts themselves suggest decomposition. An order and a shipment are different things with different rules and different lifecycles. They can be built, tested, and reasoned about independently. If your decomposition cuts across domain concepts rather than along them, you'll feel the resistance in the code.

**Behavioural boundaries.** What must happen versus what should happen versus what could happen. The happy path, the error paths, and the edge cases are naturally separable. Building and verifying the happy path first, then adding error handling, then edge cases, is a decomposition that produces incremental, demonstrable progress.

**Dependency boundaries.** What depends on what? A piece of work that has no dependencies on other in-progress work can proceed immediately. Identifying the dependency graph early reveals which pieces can be parallelised and which must be sequenced — and often reveals that some dependencies are accidental rather than necessary, which is an opportunity to restructure.

**Knowledge boundaries.** What is known and what is unknown? Novel problems — new domains, new technologies, unfamiliar requirements — carry uncertainty that established patterns don't. The right decomposition for an uncertain area often involves a spike or exploration task whose output is understanding rather than code, followed by implementation tasks whose scope is now better-defined.

---

### Vertical vs Horizontal Slicing

One of the most practically important decomposition decisions is whether to slice work horizontally or vertically.

**Horizontal slicing** divides by technical layer: do the database layer first, then the service layer, then the API, then the UI. This feels natural because it follows the technical architecture. But horizontal slices don't deliver value independently — a database schema with no service layer does nothing a stakeholder can see or validate.

**Vertical slicing** divides by behaviour, cutting through all layers for a thin slice of end-to-end functionality. The thinnest viable slice of a feature — even if it handles only the simplest case — is deployable, demonstrable, and validatable. Subsequent slices add breadth and depth.

A checkout feature sliced horizontally: design the database schema, implement the order repository, implement the use case, implement the API endpoints, implement the UI. None of these is independently valuable; all must be complete before anything works.

The same feature sliced vertically: implement end-to-end order placement for a single payment method with no discounts — working UI to working database. Then add a second payment method. Then add discount codes. Then add address validation. Each slice works, each can be demonstrated, and each delivers incremental value.

Vertical slicing is harder because it requires understanding the full stack and the full problem before starting. It also produces much better outcomes — faster feedback, earlier validation, smaller PRs, and a system that is always in a working state rather than perpetually almost working.

---

### Decomposition and TDD

TDD and decomposition are deeply connected. Writing a failing test before implementation is an act of decomposition — it requires you to specify the behaviour you want, define the interface the implementation will present, and identify the smallest increment that represents meaningful progress.

The red-green-refactor cycle is a decomposition rhythm. Each red-green cycle is a tiny problem — make this assertion true — decomposed from a larger problem. The discipline of keeping each cycle small forces continuous decomposition. A test that is hard to write is often a signal that the decomposition is wrong — the unit under test is too large, too coupled, or too unclear in its purpose.

Working test-first on a new use case reveals the decomposition naturally. You start with the simplest case, get it green, then ask: what's the next most important behaviour? That question, repeated, walks you through the problem in order of importance rather than in order of implementation convenience.

---

### Decomposition in the Codebase

The decomposition skill operates at the code level too. Every function, class, and module represents a decomposition decision — a choice about what belongs together and what belongs apart.

**Functions** should do one thing at one level of abstraction. A function that validates input, transforms it, persists it, and sends a notification is four things. Each of those is a natural decomposition boundary. The tell is when naming the function requires "and" — `validateAndSave`, `processAndNotify`. The "and" is a seam.

**Classes** should have a single reason to change. A class whose behaviour needs updating both when the business rules change and when the database schema changes has two concerns that haven't been decomposed. The Clean Architecture layer boundaries are decomposition decisions applied at architectural scale — the domain layer changes for domain reasons, the infrastructure layer changes for infrastructure reasons, and those concerns never bleed across.

**Parameters and data structures** reveal decomposition quality. A function that takes eight parameters probably contains a concept that hasn't been named and extracted. Three of those parameters belong to an `Address`, two belong to a `PaymentDetails`, and once those are named, the function signature becomes coherent.

---

### When Decomposition Is Hard

Some problems genuinely resist decomposition, and it is worth recognising why.

**The problem is not yet understood.** You can't decompose something you don't understand. The right response is exploration — a spike, a prototype, a conversation with a domain expert — before decomposing. Decomposing prematurely, before the problem's structure is visible, produces arbitrary divisions that will be wrong in ways that only become apparent later.

**The decomposition requires a decision that hasn't been made.** Sometimes a piece of work is blocked not on other work but on a design or product decision. Surfacing that dependency explicitly — "we can't decompose this further until we decide X" — is more useful than attempting a decomposition that assumes an answer.

**The existing code makes clean decomposition hard.** Debt in the existing codebase can impose its own structure on new work, making it difficult to slice cleanly. Recognising this — and sometimes paying down that debt as a prerequisite to clean decomposition — is part of the skill. A piece of new behaviour that should be a clean vertical slice becomes entangled with existing structure that needs untangling first.

**The pieces are genuinely interdependent.** Sometimes the coupling is real and not accidental. Some problems have a core that must exist before anything else makes sense. The skill here is recognising the minimal core — the irreducible piece that everything else depends on — and building that first, cleanly, so that subsequent pieces can be built against it independently.

---

### The Relationship to Everything Else

Decomposition is the skill that makes the other practices work. CI requires integrating small increments — which requires decomposing work into small increments. Clean Architecture creates natural seams — but recognising and exploiting those seams requires decomposition skill. TDD drives design through small behavioural steps — which is decomposition applied at the test level. Small PRs, fast review, confident refactoring — all of these depend on work being decomposed into pieces that are coherent, independent, and small enough to be manageable.

It is also the skill that is hardest to teach directly and most reliably developed through practice and feedback. Reviewing someone else's decomposition — asking why this boundary rather than that one, noticing where the pieces aren't as independent as they appear, suggesting a vertical slice where a horizontal one was planned — is one of the highest-value things a senior developer can do in a mentoring relationship. The feedback loop on decomposition decisions is often too slow for the lesson to land naturally. Explicit, timely attention to how work is being broken down accelerates the development of a skill that underpins almost everything else.
