---
title: "Technical Debt"
layout:     page
draft: false
description: "Provides an honest framework for thinking about technical debt, categorizing different types of debt and offering strategies for when to address it based on interest rates and delivery impact."
---

## Technical Debt — Thinking About It Honestly

### The Metaphor and Its Limits

Ward Cunningham coined the term "technical debt" in 1992 to describe a specific situation: shipping code that isn't quite right in order to learn something from real users, with the intention of going back to refactor once that learning has happened. The debt metaphor was deliberate — like financial debt, it accrues interest. The longer you leave the suboptimal code in place, the more every subsequent change that touches it costs.

The metaphor has since been stretched to cover almost any code someone doesn't like, which has made it less useful. Not all technical debt is the same, it doesn't all accrue interest at the same rate, and treating it as a single category leads to poor decisions about when to address it.

A more honest starting point is to distinguish between debt that was taken on deliberately and debt that accumulated without a conscious decision.

---

### Types of Debt Worth Distinguishing

**Deliberate, prudent debt** — Cunningham's original meaning. You know the better approach but choose the faster one now, consciously, because the value of learning or shipping outweighs the cost of the cleaner solution. "We'll ship this with a hardcoded config to validate the feature, then make it properly configurable once we know it's worth keeping." This is legitimate. The debt is known, bounded, and has a clear trigger for repayment.

**Deliberate, reckless debt** — knowingly cutting corners without a plan to address them, typically under schedule pressure. "We don't have time to do this properly." Sometimes unavoidable, but frequently a false economy — the interest starts accruing immediately and the repayment trigger never arrives.

**Inadvertent debt** — the code was written without knowing it was suboptimal. The team didn't understand the domain well enough yet, the patterns weren't established, or the design decision seemed right at the time and turned out not to be. This is normal and inevitable. It's not negligence — it's the cost of learning.

**Bit rot** — code that was fine when written but has become problematic as the surrounding system evolved. An abstraction that made sense for three use cases becomes a liability at thirty. A synchronous call that was fine at low volume becomes a bottleneck at scale. Nobody did anything wrong; the world changed.

The distinction matters because the appropriate response differs. Deliberate debt has a known owner and a clear remediation plan. Inadvertent debt and bit rot require discovery before they can be addressed. Reckless debt is usually a signal about a process or culture problem, not just a code problem.

---

### Interest Rates Vary

Not all debt accrues interest at the same rate. Some debt sits quietly in a rarely-touched corner of the codebase and costs almost nothing to leave in place. Some debt sits at the centre of the system and slows down every change that touches it.

The interest rate on a piece of debt is roughly proportional to two things: **how central the code is** and **how frequently it changes**. A messy implementation of a stable, rarely-touched utility function costs very little. A messy abstraction at the core of a domain that changes frequently costs enormously — every new feature requires fighting the existing structure, every bug fix risks introducing another, and onboarding a new developer to that area takes weeks instead of days.

This is why the instinct to address debt by age — "this code is old, let's rewrite it" — is often wrong. Age is a weak signal. Centrality and change frequency are the right signals.

A useful mental model: plot your codebase areas on two axes — how often does this change, and how much does working in it cost per change. The quadrant with high change frequency and high per-change cost is where debt is actively harmful and demands attention. The quadrant with low change frequency and low per-change cost can be left indefinitely without meaningful impact.

---

### When to Pay It Down

**When you're about to work in the area anyway.** The boy scout rule — leave the code better than you found it — applied judiciously. If a feature or bug fix is going to touch a piece of debt, cleaning it up in the same PR or the one immediately before is cheap. The context is already loaded, the tests are already running, and the improvement is incremental rather than a separate effort. This is the most efficient form of debt repayment and requires no special process — just the discipline to clean up as you go.

**When the interest rate has become the dominant cost.** If your velocity in a particular area has noticeably degraded — bugs are frequent, estimates are consistently wrong, developers avoid touching it — the debt is no longer background noise. It's the main story. At this point the cost of the debt is measurable in delivery terms, which makes the case for addressing it concrete rather than abstract.

**When a significant new capability is planned for the area.** Building on a weak foundation multiplies the cost of everything you add. If a major feature is planned for a module carrying significant debt, addressing the debt first is cheaper than building the feature in spite of it and carrying compounded debt into the future.

**When the team's confidence has eroded.** A codebase that people are afraid to change has a hidden cost that doesn't show up in estimates — risk aversion leads to smaller changes, more conservative solutions, and slower progress across the board. Restoring confidence in a core area has a multiplier effect on everything the team subsequently builds there.

---

### When to Accept It

**When the code is stable and isolated.** If a piece of suboptimal code hasn't needed to change in two years and is unlikely to change in the next two, the interest rate is effectively zero. Rewriting it is a cost with no return. Leave it alone.

**When the domain is still being discovered.** Refactoring toward a clean abstraction requires knowing what the right abstraction is. In a new or rapidly evolving domain, you often don't know yet. Premature cleanup can produce a clean version of the wrong thing — which is harder to change than messy code because it projects false confidence. Sometimes the right move is to let the domain stabilise before investing in the design.

**When the cost of change outweighs the benefit.** A full rewrite of a core module is a large, risky project. The benefit needs to be proportionate. If the debt is real but not severe, and the rewrite carries meaningful risk of regression or extended delivery disruption, accepting the debt and working around it is sometimes the honest choice. This isn't laziness — it's a genuine cost-benefit assessment.

**When it's debt you can contain.** Sometimes the right response to a poor abstraction isn't to fix it but to encapsulate it — wrap it behind a clean interface so that the ugliness is isolated and its blast radius is limited. The debt is still there, but it's no longer contagious. New code is written against the clean interface; the messy internals are contained.

---

### The Honest Conversation

The most common dysfunction around technical debt isn't technical — it's conversational. Developers describe debt in technical terms that don't resonate with stakeholders. Stakeholders push back because "rewrite the service layer" doesn't map to user value. Both sides leave the conversation frustrated.

The framing that works is outcomes, not code. Not "the repository layer has poor abstraction boundaries" but "every time we add a new data source — which we're doing three times this quarter — it takes twice as long as it should and introduces bugs we don't catch until production." Not "we have high coupling between the domain and infrastructure" but "we can't test this code without a running database, which means our test suite takes forty minutes and developers skip running it locally."

When debt is described in terms of its concrete effects on delivery — speed, reliability, risk, cost of change — it becomes a business conversation rather than a technical one. The stakeholder can weigh those effects against competing priorities and make an informed decision. That's the right conversation to be having.

It also requires developers to be honest in the other direction — not using "technical debt" as a catch-all justification for work they find more interesting, or as a way to avoid accountability for past decisions. If a piece of debt was created deliberately and the delivery it enabled was real, that was a legitimate trade. Owning that history rather than treating all debt as something that happened to the codebase builds credibility with stakeholders.

---

### Debt as a Management Practice

The teams that handle technical debt best treat it explicitly rather than hoping it gets addressed organically. Some practical approaches:

**Make it visible.** A backlog of known debt items, with notes on where they are, what they cost, and what would trigger addressing them, prevents debt from being out of sight and out of mind. It also enables honest conversations about prioritisation rather than surprise escalations when an area becomes unworkable.

**Allocate capacity deliberately.** The "20% time for tech debt" rule is crude but not wrong in principle. If debt repayment has no protected capacity, it will always be displaced by feature work. The exact percentage matters less than the commitment — some fraction of every sprint or cycle is explicitly reserved for improvement work that doesn't map to a user story.

**Attach debt work to feature work.** Rather than scheduling large separate "debt sprints" — which are demoralising, often underscoped, and leave the team feeling like they're spinning wheels — attach cleanup to the feature work that touches the same area. The context is shared, the benefit is immediate, and the progress is visible.

**Don't let the backlog become a dumping ground.** A debt backlog with 200 items is not a management tool — it's a guilt list. Items that have been there for two years without ever becoming prioritised enough to address should be closed with a note. If the debt hasn't mattered enough to fix in two years, it probably isn't debt worth tracking.

---

### The Underlying Honesty

The deepest problem with how teams talk about technical debt is a reluctance to be honest about the choices that created it. Debt accumulated under genuine time pressure is different from debt accumulated through inattention. Debt that was a good trade at the time is different from debt that was negligence dressed up as pragmatism.

That honesty matters because it points to the right fix. If the debt came from a process that consistently sacrifices quality for short-term speed, the fix is the process. If it came from a team that didn't yet understand the domain, the fix is investment in understanding before building further. If it came from a deliberate decision that was right at the time, the fix is simply repayment — no blame required.

A codebase accumulates debt the way any complex system accumulates entropy. The goal isn't a debt-free codebase — that's not achievable over a long-lived system's lifetime. The goal is debt that is known, understood, and managed — accruing where the trade is worth it, being repaid where the interest has become the dominant cost, and never allowed to grow so large that it defines what the team is capable of building.

---

## When time pressure is a thing

This is where the honest thinking about technical debt gets tested in practice. Time pressure is real, stakeholders are real, and "we should do this properly" can sound like an excuse when a deadline is looming. But the decisions made under time pressure are exactly the ones that compound — for better or worse — over the life of the system.

---

### First, Challenge the Premise

Before accepting the time pressure as fixed, it's worth asking where it comes from. Not to be obstructionist, but because the nature of the deadline changes the right response.

**Is the deadline real or inherited?** Many deadlines are estimates that became commitments that became fixed dates through a game of telephone. The original estimate had assumptions baked in that have since changed, but the date hasn't. If a deadline was set before the scope was understood, it deserves to be revisited with current information — not necessarily moved, but at least examined.

**What happens if it slips?** Some deadlines are genuinely hard — a regulatory change takes effect on a specific date, a conference has a fixed slot, a contract has a penalty clause. Others are soft — someone said a quarter and it became a plan. Understanding the actual consequence of missing the date changes the risk calculation significantly.

**Is it a scope problem or a time problem?** The most useful response to time pressure is often to negotiate scope rather than quality. Delivering a smaller thing properly is usually better than delivering a larger thing badly. "We can hit that date if we defer these three features" is a more honest conversation than silently cutting corners to fit everything in.

This isn't always possible. Sometimes the scope is fixed, the deadline is real, and you have to make a call. But getting there explicitly — through a genuine conversation about trade-offs — is better than assuming there's no flexibility and making unilateral quality trade-offs in silence.

---

### If the Pressure is Real, Be Deliberate About the Debt

Assuming the deadline is genuine and the scope is fixed, the right response is not to abandon quality practices wholesale — it's to take on debt deliberately rather than accidentally.

The distinction from the debt piece applies directly here. Deliberate, prudent debt is a conscious trade with a known cost and a plan to address it. Reckless debt is corners cut without acknowledgement, which compounds silently.

Concretely, that means:

**Name the shortcuts.** When you make a trade-off — hardcoding something that should be configurable, skipping an abstraction that would take too long to get right, writing a test that covers the happy path but not the edge cases — write it down. A comment in the code, a ticket in the backlog, a note in the PR. The act of naming it does several things: it keeps it visible, it signals to the next developer that this was a known trade rather than an oversight, and it forces you to be honest about whether the trade is actually worth making.

**Take debt in the outer layers, not the inner ones.** Clean Architecture gives you a useful guide to where debt is safer to carry. Cutting corners in a controller or an adapter is relatively low-risk — those are thin layers with limited logic, and they're easy to replace. Cutting corners in the domain model or the core business logic is high-risk — that's the layer everything else depends on, and debt there has the highest interest rate. Under time pressure, protect the core and be pragmatic about the edges.

**Keep the tests for the domain.** The temptation under pressure is to skip tests entirely. A more defensible approach is to be selective: keep the tests that cover business rules and invariants — the things that must be correct — and defer the tests that cover less critical paths. A domain model with solid test coverage and a controller without tests is a much better position than a feature with no tests at all.

**Don't break the build.** Whatever else gets deferred, the CI pipeline stays green. Checking in failing tests, disabling checks, or bypassing the pipeline under pressure is almost never worth it. It signals to the whole team that the standards are optional under stress, which is exactly when standards matter most.

---

### The Danger of the False Economy

The most common failure mode under time pressure is the belief that cutting quality now saves time. It rarely does, and it's worth being direct about this with stakeholders when the conversation allows.

Shipping something fragile doesn't end the work — it shifts it. The bugs that weren't caught by tests that weren't written show up in production and create urgent reactive work that displaces the next planned delivery. The missing abstraction means the next feature in the same area takes twice as long. The hardcoded configuration becomes a support burden. The time "saved" is borrowed, not saved, and it's borrowed at a high interest rate.

This is the argument to make to a stakeholder who is pushing for speed over quality — not as a technical principle but as a delivery reality. "If we skip the tests on this to hit the date, we're likely to spend the first two weeks of next quarter on bugs rather than the next feature." That's a business argument, not a technical one, and it's usually true.

---

### After the Pressure Passes

The moment the deadline is met is the highest-risk moment for the debt that was taken on to get there. There is a natural pressure to move immediately to the next thing — the team is stretched, there's a backlog of deferred work, and the system is running fine for now.

This is exactly when the debt should be addressed, for two reasons. First, the context is still fresh — the team knows what was cut, where the rough edges are, and what needs doing. That knowledge fades quickly. Second, the system is now in production and real usage will reveal which of the shortcuts matter. Addressing debt while the context is live is significantly cheaper than addressing it six months later when nobody remembers why the code looks the way it does.

A practical commitment: immediately after a high-pressure delivery, before moving to the next feature, spend a defined period — a few days, a sprint — addressing the most significant debt items from the push. Treat it as part of the delivery cost, not a separate discretionary project. The team that delivered under pressure should be the team that cleans up after it, while the knowledge is still there.

---

### The Culture Question

Repeated time pressure that consistently produces quality trade-offs is not a project management problem — it's a culture problem. If every quarter ends in a crunch, if technical debt tickets never get prioritised, if the answer to every scope conversation is "we need all of it by the date", the individual decisions about what to cut start to matter less than the system producing them.

In that situation, the honest thing is to name the pattern and its consequences explicitly — to leadership, in retrospectives, in planning conversations. Not as complaint but as data: here is what the last three crunch periods cost us in the following quarter, here is the trend in our delivery velocity, here is the relationship between the shortcuts we took and the bugs we're now fixing.

Teams that normalise cutting corners under pressure don't get faster over time — they get slower, as the accumulated debt raises the cost of every subsequent change. Making that visible is a leadership responsibility, and sometimes the most important technical contribution a senior developer can make is refusing to let the pattern go unnamed.
