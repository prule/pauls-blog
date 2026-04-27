---
title: "Technical Documentation — What's Worth Writing Down"
layout:     page
draft: false
---

## Technical Documentation — What's Worth Writing Down

### The Documentation Paradox

Documentation has a reputation problem in both directions. Teams that write too little leave critical knowledge locked in individuals' heads, where it fades and eventually disappears when people move on. Teams that write too much produce documents that are immediately out of date, never read, and actively misleading — giving readers false confidence that they understand something they don't.

The failure mode is treating documentation as a uniform activity — either you document everything or you document nothing. The more useful question is: what specific knowledge needs to exist in written form, for whom, and why? Different answers produce radically different kinds of documentation, each with different value and different maintenance costs.

The test that cuts through most documentation debates is simple: if this information were lost, what would the cost be, and could it be recovered from the code alone? Information that is costly to lose and can't be recovered from the code is worth writing down. Information that is easily recoverable from the code, or cheap to reconstruct, probably isn't.

---

### What the Code Cannot Say

Code is an excellent record of what the system does. It is a poor record of why it does it that way, what alternatives were considered, what constraints shaped the design, and what assumptions underpin it. These are the gaps that documentation fills.

**The why behind non-obvious decisions.** A piece of code that looks wrong but isn't — because it's working around a specific platform bug, meeting a regulatory requirement, or compensating for a known limitation in a dependency — will be "fixed" by the next developer who encounters it unless the reason is written down. The comment or document that explains the why prevents the fix that reintroduces the problem.

This is the highest-value documentation and the most neglected. It requires almost no maintenance — the why behind a decision doesn't change when the code changes, unless the decision itself changes. And its absence has a specific, recurring cost: the rediscovery of knowledge that already existed, paid by every developer who encounters the code without the context.

```kotlin
// Without context — looks like an off-by-one error, will be "fixed"
val retryDelay = Duration.ofSeconds(31)

// With context — the constraint is visible, the fix doesn't happen
// Payment gateway requires a minimum 30-second gap between retry attempts
// per their API contract (section 4.2). Using 31 seconds to avoid
// edge cases around clock synchronisation between our systems.
val retryDelay = Duration.ofSeconds(31)
```

**Constraints and assumptions.** Every design makes assumptions about the environment it operates in — expected load, data characteristics, integration partner behaviour, operational context. Those assumptions are invisible in the code but critical for anyone making changes. When assumptions are violated — because the system is used in a way the original author didn't anticipate — the result is subtle bugs and unexpected behaviour. Writing the assumptions down makes them explicit and testable.

**The decisions not made.** An architecture decision record (ADR) that only records the decision made is less valuable than one that also records the options considered and rejected. The developer who later asks "why didn't they just use X" — and X was considered and rejected for a good reason — will make the same mistake again unless the rejection is documented. The record of the road not taken is often more valuable than the record of the road taken.

---

### Architecture Decision Records

ADRs deserve specific attention because they address the most common and most costly documentation gap: the loss of architectural context over time.

An ADR is a short document that records a significant architectural decision. The format is deliberately lightweight — not a formal specification but a record of the thinking at a point in time. A minimal ADR contains:

**Context** — what was the situation that made this decision necessary? What problem were we trying to solve? What constraints were we operating under?

**Options considered** — what alternatives were evaluated? What were the trade-offs of each?

**Decision** — what was chosen, and why?

**Consequences** — what does this decision imply? What does it make easier? What does it make harder? What assumptions does it rest on?

ADRs are stored in the repository alongside the code — typically in a `/docs/decisions` directory — so they version with the code and are visible to anyone working in the codebase. They are written at decision time, when the context is fresh, not retrospectively when the context is gone.

The discipline is identifying what constitutes a significant decision. Not every design choice warrants an ADR. The threshold is roughly: would a new team member, encountering this aspect of the system, be likely to question it or want to change it? If the answer is yes, the decision is worth recording. If the design choice is obvious from the code and the domain, it isn't.

---

### Operational Documentation

Code that runs in production has an operational dimension that the code itself cannot capture. The people who operate the system — who respond to incidents, manage deployments, and monitor health — need information that no amount of reading the source will provide.

**Runbooks** — step-by-step procedures for specific operational scenarios. How to perform a database migration. How to roll back a deployment. What to do when the queue depth exceeds a threshold. What the likely causes of specific alerts are and how to investigate them. Runbooks don't need to be elaborate — a short document with numbered steps is often enough. Their value is in reducing the cognitive load on someone dealing with an urgent situation, where having the steps written down is the difference between a ten-minute resolution and an hour of investigation.

**System dependencies and integration points.** What external systems does this service depend on? What are the failure modes if each dependency is unavailable? What retry and fallback behaviour is in place? This information is often buried in the code or entirely undocumented. A simple diagram or document that maps the integration points gives operators the context to understand incident blast radius and to make informed decisions under pressure.

**Known issues and limitations.** Every system has known quirks — behaviours that are surprising but intentional, limitations that exist for understood reasons, edge cases that are handled imperfectly. Documenting these prevents the same investigation being done repeatedly by different people, and prevents well-intentioned changes that try to fix something that cannot be fixed without broader consequences.

---

### What Tests Document

Tests are the most reliable form of documentation because they are executed — they cannot drift from the code they describe without failing. A test suite that is expressive and well-named is a specification of the system's behaviour that stays current automatically.

This means the test suite carries significant documentation responsibility. Tests named in terms of behaviour — `when a customer places an order with insufficient stock, the order is rejected with a clear error` — communicate intent that a reader can understand without running the code. Tests named `test1`, `testOrderError`, or `shouldWork` communicate nothing.

The implication is that test quality is a documentation concern as much as a correctness concern. Vague test names, tests that exercise multiple behaviours simultaneously, test data that doesn't reveal the relevant preconditions — these reduce the documentation value of the test suite without reducing its coverage value. Writing tests as specifications — clear preconditions, clear actions, clear expected outcomes, meaningful names — produces a suite that serves both purposes.

---

### API Documentation

Interfaces exposed to other teams or external consumers are a special case. The consumer of an API needs to understand what it does without reading its implementation. This is the one area where documentation is genuinely non-optional — without it, the API cannot be used confidently.

Good API documentation covers: what the endpoint does in business terms, what each parameter means and its valid range, what the response represents, what error conditions exist and what they mean, and what the caller should do in each error case. Examples are worth more than descriptions — a concrete request and response pair communicates what prose often doesn't.

The discipline is keeping API documentation close to the code that implements it — ideally generated from annotations in the code rather than maintained as a separate document. Separate documents drift; documentation embedded in the code is at least in the same place as the thing it describes, which makes it more likely to be updated when the code changes.

---

### What Isn't Worth Writing Down

Documentation has a maintenance cost. Every document that exists must be kept current, or it becomes actively misleading — worse than no documentation, because it gives the reader false confidence. The cost of maintenance means that documentation which doesn't carry commensurate value is a liability.

**What the code already says clearly.** A comment that restates what the code does in prose adds no information and doubles the maintenance burden. When the code changes, the comment must change too — and it frequently doesn't, which produces a comment that contradicts the code it describes.

```kotlin
// Bad — restates the code, will drift out of sync
// Iterate over orders and calculate the total
val total = orders.sumOf { it.total }

// Good — the code says what it does; no comment needed
val total = orders.sumOf { it.total }
```

**Process documentation that lives outside the process.** A development process documented in a wiki page that nobody reads is not a process — it's archaeology. If a process is worth following, it should be enforced or at least visible in the tools people actually use. A PR template that prompts for test coverage is more effective than a document that describes the policy. Automated checks are more effective than written guidelines.

**Speculative documentation.** Documentation written in anticipation of questions that may never be asked, or explaining features that may never be built, consumes writing time and creates maintenance overhead without providing value. The best documentation is written in response to a demonstrated need — a question that was asked, a confusion that arose, a decision that needs to be recorded.

**Documentation that will immediately be out of date.** Step-by-step instructions for processes that change frequently are often more costly than valuable. By the time someone follows them, they may be wrong. Living documentation — generated from the system, validated against the system, or embedded in the tools — ages better than narrative documentation.

---

### The Maintenance Problem

The fundamental challenge of documentation is that it is written at a point in time and the system continues to change. Every change to the system is a potential documentation debt — somewhere a document may now be wrong, and the person who made the change may not know the document exists.

Several practices reduce this problem:

**Keep documentation close to the code it describes.** Documentation in the repository, in the same file as the code, in adjacent files — this is more likely to be noticed and updated when the code changes than documentation in a separate wiki. The principle is co-location: the further documentation is from the code it describes, the faster they diverge.

**Prefer documentation that is generated or validated automatically.** API documentation generated from code annotations, architecture diagrams generated from the actual deployment configuration, dependency graphs generated from the actual dependency declarations — these stay current because they are derived from the authoritative source rather than maintained separately.

**Write less, more carefully.** A small amount of high-quality, well-maintained documentation is more valuable than a large amount of stale documentation. Choosing carefully what to document — applying the test of what is costly to lose and can't be recovered from the code — produces a smaller documentation surface that is more likely to stay current.

**Treat outdated documentation as a bug.** When a document is found to be wrong, fixing it should be treated with the same seriousness as fixing incorrect code. A culture that tolerates stale documentation is one where documentation cannot be trusted — which means it stops being read, which means it stops being maintained, which accelerates the staleness. The cycle runs in both directions.

---

### The Right Amount

The right amount of documentation is the minimum that ensures critical knowledge doesn't live only in people's heads. Not a byte more, because every additional byte has a maintenance cost. Not a byte less, because the lost knowledge has a recovery cost.

That minimum is higher than most teams maintain and lower than most documentation advocates recommend. It consists largely of the why behind non-obvious decisions, ADRs for significant architectural choices, operational runbooks for complex procedures, and expressive tests that serve as executable specification. Everything else should be justified by a specific demonstrated need rather than added speculatively.

The team that writes this minimum carefully, keeps it close to the code, and treats its maintenance as seriously as the code itself will have documentation that is trusted, read, and genuinely useful. That is rarer than it should be, and more valuable than most teams realise until the knowledge it would have preserved is gone.
