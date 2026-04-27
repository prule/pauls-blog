---
title: "Continuous Integration and Delivery"
layout:     page
draft: false
---

## Continuous Integration and Delivery

### The Problem with Infrequent Integration

Imagine two developers working independently on the same codebase for two weeks. They've each made hundreds of changes across dozens of files. On day fourteen they attempt to merge. The merge itself might be mechanical — resolving conflicting edits line by line — but the harder problem is behavioural: their changes made assumptions about the system that are no longer true. Tests fail in ways that are hard to diagnose because the surface area of change is enormous. Nobody is sure whether a failure was introduced yesterday or ten days ago.

This is **integration hell** — and it was the normal state of software development for decades. Release cycles measured in months meant integration events measured in days of pain. The cost was accepted as inevitable.

It isn't inevitable. It's a consequence of batch size.

---

### What Continuous Integration Actually Means

Continuous Integration (CI) means every developer integrates their work into the shared mainline **frequently** — at minimum daily, ideally several times a day. Each integration is verified by an automated build and test suite. If the build breaks, fixing it is the team's immediate priority.

This is the original definition from Kent Beck's Extreme Programming in the late 1990s, later elaborated by Martin Fowler. It is worth stating clearly because "CI" has been diluted — many teams use the term to mean "we have a build pipeline" without practising the frequent integration that makes the pipeline valuable. A pipeline that runs against long-lived feature branches is not CI. It is automated building of isolated work, which is useful but solves a different problem.

True CI has a few non-negotiable properties:

- **A single mainline** that represents the current state of the system
- **Frequent commits to that mainline** — not to personal branches that merge weeks later
- **A fast automated build** that verifies every integration
- **A shared commitment** to keeping the build green — a broken build is not a background condition, it is an emergency

---

### Why Small Batches Change the Economics

The insight at the heart of CI is that integration cost is not linear in batch size — it's superlinear. Integrating one day of work takes minutes. Integrating two weeks of work takes hours or days, and the cost continues to grow as the batch grows. Bugs compound. Assumptions collide. Context is lost.

Small, frequent integration keeps each integration event cheap. The cost is paid continuously in small increments rather than periodically in large painful episodes. Total integration cost over a project lifetime falls dramatically.

This changes what is economically viable. When integration is expensive, you defer it — which makes it more expensive, which causes you to defer it further. When integration is cheap, you do it constantly, which keeps it cheap. The two approaches create self-reinforcing cycles in opposite directions.

The same logic applies at every level of the delivery pipeline:

| Practice | Batch being reduced | Cost being smoothed |
|---|---|---|
| Frequent commits | Code changes | Merge conflicts, integration bugs |
| CI pipeline | Build and test verification | Bug detection latency |
| Continuous delivery | Release preparation | Deployment risk |
| Feature flags | Feature exposure | User impact of changes |

Each practice reduces batch size at a different level, and each produces the same economic shift — from large infrequent costs to small continuous ones.

---

### The CI Pipeline

The pipeline is the automated system that verifies each integration. A well-designed pipeline is fast, reliable, and comprehensive enough that a green build is genuinely meaningful.

A typical pipeline in order of execution:

**Compile / build** — the fastest possible check. Does the code even assemble? Catches syntax errors, missing dependencies, type errors in compiled languages. Should complete in seconds.

**Unit tests** — fast, isolated tests covering individual components. Should complete in under a minute for most codebases. These run first because they're cheapest and catch the most common failures.

**Static analysis / linting** — code style, complexity thresholds, common error patterns. Keeps the codebase consistent without requiring human review for mechanical issues.

**Integration tests** — tests that exercise real dependencies: databases, message queues, external service stubs. Slower than unit tests, but verify that components connect correctly.

**End-to-end tests** — a small suite covering critical paths through the full system. Slowest and most brittle; kept small deliberately.

**Security scanning** — dependency vulnerability checks, static security analysis. Fits naturally into the pipeline and catches issues without manual effort.

The total pipeline time matters. A pipeline that takes forty minutes provides slow feedback — developers have moved on to other work by the time results arrive, context has been lost, and the temptation to push without waiting grows. A pipeline under ten minutes keeps feedback tight. Under five minutes is better. The pipeline is a tool for the team; it should be treated as a performance-sensitive system and optimised accordingly.

---

### Pull Requests and the Review Bottleneck

The pipeline can be fast, the tests reliable, and the deployment automated — but if human review is a bottleneck, the batch size grows back and the economics revert. A PR that sits for two days waiting for review has the same effect as infrequent integration: context is lost, work piles up behind it, and the developer has moved on to something else by the time feedback arrives.

This is one of the most common failure modes in teams that try to adopt CI. Fixing the pipeline without fixing the review culture is solving half the problem.

**PRs must be small.** A 1,200-line PR is not reviewable in the same way an 80-line PR is. Large PRs get deferred because reviewers don't have a contiguous block of time to do them justice. A PR that can be understood in five minutes gets reviewed immediately. A PR that requires thirty minutes of study gets scheduled — and scheduling means waiting.

This means decomposing work differently. Instead of a PR that implements an entire feature, a series of PRs that each do one coherent thing. A new feature might arrive as: the domain model and its tests, then the use case, then the repository implementation, then the controller. Each is reviewable in isolation, mergeable independently, and safe to integrate even though the feature isn't yet complete.

**Review agreements need to be explicit.** "Review PRs promptly" means different things to different people. Agreeing on a specific expectation removes ambiguity — PRs under a certain size get reviewed within two hours, or there are two fixed review slots per day that everyone honours. The agreement should be treated like the broken build rule: a collective commitment, not a guideline individuals follow when convenient.

**Not every change needs the same process.** Applying the same review overhead to a one-line config change and a new authentication flow treats equal risk unequally. Trivial changes — dependency updates, documentation, configuration — can move faster. Substantive changes get full review. High-risk changes — schema migrations, security-sensitive code, public API changes — warrant synchronous review or pairing. Calibrating the process to the risk keeps flow fast for routine work while maintaining rigour where it matters.

**Pairing changes the review dynamic.** If two people worked on something together, the formal PR review becomes confirmation rather than discovery. The second person already has context, has seen the approach, and has influenced the outcome. Some teams formalise this: code that was genuinely paired on doesn't require traditional review. The collaboration was the review. This removes the waiting entirely and creates a natural incentive for the collaborative patterns that produce better code anyway.

---

### How Clean Architecture Enables Small PRs

The ability to decompose a feature into small, independently mergeable PRs isn't just a matter of discipline — it depends heavily on how the codebase is structured. Clean Architecture creates the natural seams that make this decomposition possible.

Because layers are explicitly separated and dependencies only point inward, each layer of a feature can be built and merged without the others being present. The domain model is pure business logic with no infrastructure dependencies — it can be merged the moment it's coherent and tested, before a database or HTTP layer exists. The use case depends only on the domain and repository interfaces — it can be merged and tested against stubs before the real repository implementation exists. The repository implementation slots in behind its interface without the use case needing to change. The controller is a thin adapter that wires the delivery mechanism to the use case.

Each layer has a single concern, clear boundaries, and is testable in isolation. Each maps naturally onto a PR that a reviewer can understand without holding the entire feature in their head simultaneously.

> A new feature in a well-structured codebase has a natural delivery sequence:
>
> - Add the domain model and its tests — pure Kotlin, no dependencies, trivially mergeable
> - Add the use case interface and implementation against the domain — still no infrastructure
> - Add the repository interface — a contract, no implementation yet, use case can be tested with a stub
> - Add the repository implementation — infrastructure detail, slots in behind the interface
> - Add the controller — thin adapter, wires the HTTP layer to the use case
> - Enable via feature flag
>
> Each of these is a coherent, reviewable, mergeable unit. Each PR is small because each layer has a single responsibility and clear boundaries. The reviewer's job is easier too — they're not looking at a sprawling change across multiple concerns simultaneously, they're reviewing one layer at a time with a clear mental model of what it should and shouldn't know about.

The contrast with an unstructured codebase is stark. When there are no clear layer boundaries, a feature touches everything simultaneously — controller, service, repository, entity, DTO, database migration — because they're all coupled together. That change can't be split meaningfully because each piece only makes sense in the context of all the others. It arrives as one large PR, sits in the review queue, and the reviewer faces a sprawling change across multiple concerns with no obvious place to start.

Clean Architecture also improves review quality. When a PR touches only one layer, the reviewer can apply focused questions appropriate to that layer — is this domain model protecting its invariants, does this use case do one thing, is this controller thin enough — rather than having to reason about the entire system simultaneously. Smaller cognitive surface area means faster, deeper review.

The same principle applies to the rich domain model. If business logic lives inside the domain object, the domain PR is self-contained and testable without any infrastructure present. If business logic lives in a service that also coordinates infrastructure, the logic and the infrastructure arrive together and can't be reviewed or merged independently.

The practices compound: the same architectural boundaries that make code testable and maintainable are the boundaries that make work decomposable into small, safe, independently-mergeable increments.


---

### Continuous Delivery and Continuous Deployment

**Continuous Delivery (CD)** extends CI with a further commitment: the mainline is always in a releasable state. Every integration that passes the pipeline could be deployed to production. The decision to deploy is a business decision, not a technical one.

This requires a higher standard of pipeline confidence and a higher standard of code quality. You can't have known failing tests checked in with a note to fix them later. You can't have half-finished features that break the user experience if deployed. Techniques like feature flags (deploying code that isn't yet active) and branch by abstraction (replacing a component incrementally) make this tractable.

**Continuous Deployment** goes one step further: every integration that passes the pipeline is deployed to production, automatically, without human approval. This is the practice of companies like Amazon, Netflix, and Etsy — organisations that deploy hundreds or thousands of times per day.

The distinction matters:

- CI: integrate frequently, verify automatically
- Continuous Delivery: always releasable, deploy on demand
- Continuous Deployment: always released, deploy automatically

Most organisations practice CI and aim for Continuous Delivery. Continuous Deployment requires a high degree of pipeline confidence, observability, and operational maturity — but the organisations that practise it report that deployment becomes so routine it ceases to be an event worth noting.

---

### What It Requires of the Team

CI is a technical practice with a social contract at its centre. The pipeline is worthless if people ignore a broken build, commit infrequently to avoid conflicts, or maintain long-lived branches that only touch mainline at the end of a feature.

**The broken build is an emergency.** When the build breaks, fixing it takes priority over new work. Not tomorrow. Not after the current task. Now. A broken build that persists for hours signals that the team doesn't believe in the pipeline — and a pipeline the team doesn't believe in stops being a safety net.

**Commits are small and frequent.** This is the hardest cultural change for developers used to working in isolation. It requires a different approach to task decomposition — breaking work into increments that are safe to integrate even when the feature isn't complete. This is a skill, and it takes practice.

**The pipeline is everyone's responsibility.** Flaky tests that sometimes pass and sometimes fail erode confidence in the pipeline. Every flaky test is a broken window — it teaches the team to distrust the green signal, which eventually makes the pipeline meaningless. Flaky tests are fixed or deleted, not tolerated.

---

### The Relationship with Other Practices

CI and CD don't exist in isolation — they compose with the other practices that make software development sustainable.

**TDD** makes CI viable at high frequency. Tests written test-first tend to be fast, isolated, and reliable — the properties a CI pipeline depends on. The tight red-green-refactor cycle also naturally produces small, focused commits, because each cycle is a coherent unit of work with a clear finishing point.

**Clean Architecture** makes small PRs structurally natural rather than a matter of willpower. The layer boundaries are the PR boundaries. A team working in a well-structured codebase doesn't have to fight the code to decompose their work — the seams are already there.

**Trunk-based development** is the branching strategy that CI implies. Short-lived branches — hours, not days — merged frequently to a single mainline. Feature flags handle the problem of incomplete features being present in production code without being visible to users.

**Renovate and dependency management** integrate naturally — dependency update PRs run through the same pipeline as code changes, and auto-merge on a green build is safe precisely because the pipeline is trusted.

**Observability** becomes more important as deployment frequency increases. When you're deploying multiple times a day, you need to know quickly if a deployment introduced a problem. Good metrics, logging, and alerting close the feedback loop from production back to the team.

---

### The Compounding Effect

The deepest benefit of CI and CD isn't any single practice — it's what they make possible over time.

A team that integrates and deploys frequently builds confidence. Each deployment is small; rollback is straightforward; the gap between what's in production and what's in the repository is measured in hours. This confidence enables faster iteration — you can try something, observe the result in production, and respond. The feedback loop from idea to user is short.

A team that integrates and deploys infrequently builds anxiety. Each deployment is large; rollback is complex; the gap between repository and production is measured in weeks. This anxiety slows everything down — more approval gates, more manual testing, more caution — which paradoxically makes each deployment riskier, because the batch grows while the team's confidence falls.

The economics compound. High-frequency teams get faster, safer, and more responsive over time. Low-frequency teams get slower and more fragile. CI and CD are not just engineering practices — they are the foundation of an organisation's capacity to learn and respond.
