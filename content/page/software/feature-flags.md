---
title: "Feature Flags and Trunk-Based Development"
layout:     page
draft: false
---

## Feature Flags and Trunk-Based Development

### The Problem They Solve Together

Continuous Integration demands frequent integration to mainline. Continuous Delivery demands that mainline is always releasable. These two commitments create an immediate tension: what do you do with a feature that takes two weeks to build? You can't keep it on a branch for two weeks — that defeats CI. But you can't have a half-built feature in production — that defeats CD.

Feature flags and trunk-based development solve this tension together. They are complementary practices that, combined, allow code to be integrated continuously while features are released deliberately.

---

### Trunk-Based Development

Trunk-based development is a branching strategy where all developers integrate their work into a single shared branch — the trunk, or mainline — frequently, and branches are either very short-lived or not used at all.

The rules are simple but demanding:

- There is one mainline branch that represents the current state of the system
- Branches, if used at all, live for hours or a day or two — not weeks
- Every developer integrates to mainline at least once a day
- The mainline is always in a buildable, deployable state

This is the branching model that CI actually requires. The alternative — long-lived feature branches that merge to mainline at the end of a sprint or when a feature is complete — is not CI. It is isolated development with an automated build, which is useful but doesn't deliver the integration benefits. The longer a branch lives, the more it diverges, and the more painful the eventual merge becomes.

The discomfort most developers feel about trunk-based development is real: it requires integrating incomplete work, which feels wrong. Feature flags are what make it safe.

---

### Feature Flags

A feature flag — also called a feature toggle — is a conditional in the code that controls whether a given feature is active. The code is deployed, but the feature is only exposed when the flag is enabled. Enabling the flag is a separate, independent act from deploying the code.

In its simplest form:

```kotlin
if (featureFlags.isEnabled("new-checkout-flow")) {
    renderNewCheckoutFlow()
} else {
    renderLegacyCheckoutFlow()
}
```

This decouples two things that are often conflated: **deployment** (putting code into production) and **release** (making a feature available to users). Once they are decoupled, each can be managed on its own terms.

Deployment becomes a technical event — low-risk, frequent, unremarkable. The new code is in production but dormant. Release becomes a business event — deliberate, controlled, timed to whatever the right moment is. Marketing can coordinate a launch. A gradual rollout can begin with a small percentage of users. A specific customer can get early access. The product team decides when, not the deployment schedule.

---

### The Flag Lifecycle

A feature flag is not a permanent addition to the codebase. It has a lifecycle, and managing that lifecycle is part of the discipline.

**Development** — the flag is off everywhere. Developers work behind it, integrating freely to mainline without affecting anyone. The incomplete feature is invisible in production.

**Testing and QA** — the flag is enabled in test and staging environments. The feature can be verified end-to-end without being visible to users.

**Gradual rollout** — the flag is enabled for a small percentage of production traffic, or for internal users, or for a specific cohort. Real usage is observed. Problems surface with limited blast radius.

**Full release** — the flag is enabled for everyone. The feature is live.

**Cleanup** — this is the step most teams neglect. Once a feature is fully released and stable, the flag and both code paths should be removed. The flag served its purpose; it is now dead code. A codebase littered with old flags — each wrapping a conditional, each maintaining two code paths — becomes increasingly hard to reason about. Flag debt is real debt.

---

### Types of Feature Flags

Not all flags serve the same purpose, and conflating them leads to confusion about how they should be managed.

**Release flags** — the most common type. Control the rollout of a feature in development. Short-lived by design; should be removed once the feature is fully released. This is the flag described above.

**Experiment flags** — used for A/B testing. Two code paths are compared against a metric. The winning path becomes the default and the flag is removed. Slightly longer-lived than release flags but still temporary.

**Ops flags** — control operational behaviour independently of features. A kill switch for a non-critical feature under load, a circuit breaker for an external dependency, a way to disable an expensive operation during an incident. These may be long-lived — they provide operational control and are valuable precisely because they can be toggled quickly in production without a deployment.

**Permission flags** — control access to features by user segment, subscription tier, or role. These are essentially permanent — they're a product feature, not a development practice. They belong in an entitlements or permissions system rather than a feature flag system.

Keeping these distinct matters practically. Release flags should be aggressively removed after release. Ops flags should be documented and maintained. Permission flags should be in a different system entirely. Mixing them together creates a flag system that is simultaneously cluttered with stale release flags and missing the ops flags that would actually be useful during an incident.

---

### Trunk-Based Development With Flags in Practice

The combination in practice looks like this. A developer begins work on a new feature. They create a flag — disabled by default in all environments. They commit their first changes to mainline the same day, behind the flag. The CI pipeline runs. The build passes. Nothing in production is affected.

Over the following days, more commits land on mainline. Each is integrated and verified by the pipeline. The feature takes shape incrementally without ever blocking or being blocked by other work happening in the same codebase simultaneously. Other developers can see the work in progress, understand its direction, and avoid conflicting changes.

When the feature is ready for testing, the flag is enabled in the test environment. When it's ready for a gradual rollout, the flag is enabled for a percentage of production traffic. When it's fully released, the flag is enabled everywhere — and shortly after, the flag and the old code path are removed in a cleanup PR.

The entire process — from first commit to flag removal — might span two weeks of calendar time, but mainline was never in a broken state, no branch diverged from reality, and every intermediate state was deployable.

---

### Branch by Abstraction

Feature flags handle the common case well, but they're not the right tool for everything. Large-scale structural changes — replacing a core library, migrating to a new architecture, swapping an infrastructure component — can't easily be wrapped in a conditional.

Branch by abstraction is the complementary technique for these situations. Rather than a flag, you introduce an abstraction layer over the thing being changed, make the new implementation and the old one interchangeable behind that abstraction, gradually migrate callers to the new path, and then remove the abstraction and the old implementation once the migration is complete.

```kotlin
// Step 1: introduce abstraction over the existing implementation
interface PaymentGateway {
    fun charge(amount: Money, card: CardDetails): ChargeResult
}

class LegacyPaymentGateway : PaymentGateway { /* existing code */ }

// Step 2: new implementation behind the same interface
class StripePaymentGateway : PaymentGateway { /* new code */ }

// Step 3: swap implementations — either via config or a flag
// Step 4: remove the legacy implementation and the abstraction if no longer needed
```

Each step is a small, safe, independently mergeable PR. The migration happens incrementally over multiple integration cycles. At no point is mainline broken. At no point is there a long-lived branch diverging from reality.

This is Clean Architecture's dependency inversion principle applied as a migration strategy — the same interface boundary that makes code testable also makes it replaceable without disruption.

---

### The Flag Management Question

At small scale, feature flags can be simple constants or environment variables. As the system and team grow, this becomes unmanageable — flags need to be togglable at runtime without a deployment, visible across services, and manageable by non-developers for operational purposes.

Dedicated feature flag platforms — LaunchDarkly, Unleash, Flagsmith, or equivalents — provide runtime flag management, targeting rules, gradual rollout percentages, audit trails, and SDKs for multiple languages. The investment is worth it once the team is large enough that flag state needs to be shared across services or toggled during incidents without waiting for a code change.

The flag system is infrastructure, not an afterthought. Teams that treat it as infrastructure — with proper tooling, documented conventions, and a flag cleanup discipline — get the full benefit. Teams that treat it as a collection of hardcoded booleans eventually create a codebase where nobody is sure which flags are live, which are dead, and what the current intended state of the system actually is.

---

### What This Enables

The combination of trunk-based development and feature flags changes what a team is capable of operationally.

**Separating release from deployment** means the team can deploy as often as they like — multiple times a day — without coordinating with marketing, support, or customers. Deployments become non-events. Releases remain deliberate.

**Instant rollback without redeployment** — if a released feature has a problem, the flag is turned off. Users are back to the previous experience in seconds, without waiting for a hotfix deployment to complete. The bad code is still in production but dormant; it can be fixed and re-released without urgency.

**Dark launching** — a new implementation can be run in production alongside the old one, receiving real traffic but not surfacing results to users, allowing performance and correctness to be validated against real load before the switch is made. This is particularly valuable for migrations where synthetic load doesn't reflect real usage patterns.

**Reduced merge risk** — because branches are short-lived and integration is continuous, the divergence between any developer's work and mainline at any given moment is small. Merge conflicts are rare and cheap when they occur. The integration tax is paid continuously in small amounts rather than periodically in large painful events.

---

### The Discipline Required

Trunk-based development with feature flags is more demanding than long-lived branches in several respects.

It requires the habit of thinking in small, safe increments — every commit to mainline should leave the system in a valid state. It requires the discipline of flag cleanup — released flags must be removed, not left to accumulate. It requires a CI pipeline that is fast and trusted enough that integrating to mainline multiple times a day is not a source of anxiety.

Most fundamentally, it requires a shift in how developers think about their work. The branch model lets you work in isolation until something feels finished. Trunk-based development requires integrating continuously, which means your work is visible and your intermediate states are real. This is uncomfortable at first. Over time it becomes the practice that keeps the system — and the team — coherent.
