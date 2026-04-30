---
title: "Shape Up: Stop Running in Circles and Ship Work that Matters"
description: "A summary of Shape Up by Ryan Singer, detailing Basecamp's approach to software development and shipping work that matters."
layout:     page
draft: false
---


## Shape Up: Stop Running in Circles and Ship Work that Matters — Summary

**By Ryan Singer — Head of Strategy at Basecamp**

{{< notice type="info" title="Online book" >}}
Basecamp.com [Shape Up: Stop Running in Circles and Ship Work that Matters](https://basecamp.com/shapeup)
{{< /notice >}}

---

### Core Thesis

Most software teams are trapped in a cycle that feels productive but isn't: endless backlogs, two-week sprints that never quite finish, features that balloon beyond their original scope, and a persistent sense that the team is always busy but never done. Shape Up is Basecamp's answer to this — a **fundamentally different way of thinking about how software gets built**.

The core insight: **the problem is not execution — it is the shape of the work before it reaches the team.**

> **"The way you give work to teams determines almost everything about how that work turns out."**

---

### The Three Core Activities

Shape Up organises product development into three distinct activities that happen at different levels of the organisation:

**Shaping** — figuring out what to build, done by senior people before work is assigned
**Betting** — deciding which shaped work to actually do, done at a structured six-week cadence
**Building** — executing the work, done by small autonomous teams with full responsibility

These are not sequential phases in a single project. They run in **parallel** at different stages of maturity.

---

### Part 1: Shaping

Shaping is the most original and important concept in the book. It is the work that happens **before** a feature goes to a team — and it is almost entirely absent from most development processes.

---

#### What Shaping Is Not

- It is not a detailed specification
- It is not wireframes or pixel-perfect designs
- It is not a user story with acceptance criteria
- It is not a ticket in a backlog

All of these are either too vague or too prescriptive. Too vague and the team flounders. Too prescriptive and you've done the design work without the people who will actually build it — which produces bad design and removes team autonomy simultaneously.

---

#### What Shaping Is

Shaping produces a **pitch** — a document that defines:

1. **The problem** — what are we actually solving and for whom?
2. **The appetite** — how much time is this worth? (Not how long will it take)
3. **The solution** — a rough sketch of the approach, at the right level of abstraction
4. **Rabbit holes** — known complications to avoid
5. **No-gos** — explicit things this solution will not do

The shaped work is **rough but not vague**. It gives teams enough direction to start and enough freedom to make good decisions.

---

#### The Appetite — The Most Powerful Reframe

Traditional estimation asks: *"How long will this take?"*

Shape Up asks a completely different question: *"How much time is this worth?"*

This is the **appetite** — a time budget set before design begins, based on the value of the problem, not the complexity of the solution.

Two appetite sizes:
- **Small batch** — one or two weeks of work for one or two people
- **Big batch** — up to six weeks of work for a small team

The appetite is not a deadline imposed after scoping. It is a **design constraint** that shapes what the solution can be. If a solution doesn't fit in the appetite, you either find a simpler solution or decide the problem isn't worth that much time.

This reframe is profound: instead of a feature growing to fill unlimited time, the time box forces creative constraint that almost always produces better, simpler solutions.

---

#### Breadboards and Fat Marker Sketches

Singer introduces two specific tools for shaping at the right level of abstraction:

**Breadboards** — borrowed from electronics. Sketch the key screens and components in words and simple connections — not visual design. Shows the flow and the parts without committing to any visual decisions.

**Fat marker sketches** — literally drawn with a thick marker so the level of detail is physically constrained. Shows spatial relationships and rough layout without pixel-level decisions that would be premature.

Both tools communicate intent without over-specifying implementation — leaving the right decisions to the people who will build it.

---

#### Finding the Right Level of Abstraction

One of the book's most practical skills: knowing when a shape is too abstract or too concrete.

**Too abstract:** "Improve the notifications experience" — no direction, team has to start from scratch, scope is unlimited

**Too concrete:** 47 detailed wireframes with exact copy, colours, and interaction states — team becomes implementors rather than problem solvers, design decisions made by the wrong people at the wrong time

**Just right:** "Add a way for users to mute notifications from specific projects for a set time period — think a simple overlay accessible from the notification bell, no need for a full settings page" — clear problem, clear constraint, room for the team to design the actual solution

---

### Part 2: Betting

---

#### The Betting Table

Every six weeks, Basecamp holds a **betting table** — a small meeting of senior stakeholders who decide which shaped pitches will be worked on in the next cycle.

This is deliberately unlike:
- Sprint planning — which processes a backlog of pre-approved work
- Roadmap reviews — which commit to delivery dates months in advance
- Prioritisation meetings — which rank an ever-growing list

The betting table starts **fresh every cycle**. Nothing is automatically carried forward. Every pitch must make its case on its own merits right now.

---

#### No Backlog

This is the most radical operational decision in the book. **Basecamp has no backlog.**

Singer's argument: backlogs are where good ideas go to slowly become irrelevant while creating the illusion of captured value. A backlog grows indefinitely, requires constant grooming, demoralises teams with its endless incompleteness, and creates pressure to work on old ideas instead of responding to current reality.

Instead: if an idea is worth doing, it will be worth pitching again when the time is right. If it's not worth pitching again, it wasn't worth doing. The discipline of reshaping and re-pitching ensures only genuinely current and valuable work gets done.

> **"If it's not worth someone's time to shape it into a pitch, it's not worth the team's time to build it."**

---

#### The Six-Week Cycle

Six weeks is long enough to build something meaningful and short enough to maintain urgency and prevent drift. It is not arbitrary — Singer specifically argues against both shorter and longer cycles:

- **Two-week sprints** are too short to build anything substantial, creating constant context switching and planning overhead
- **Quarterly cycles** are too long, losing the forcing function of a real deadline

Six weeks hits a specific sweet spot: a team can tackle a genuinely hard problem and ship something real, while leadership retains frequent decision points about what to work on next.

---

#### Cooldown

Between every six-week cycle is a **two-week cooldown** period. No scheduled work. Teams use this time to:

- Fix bugs they noticed during the cycle
- Explore ideas they didn't have time to investigate
- Write documentation
- Do technical tidying
- Simply rest and recover

This is not slack in the schedule to be filled. It is a **deliberate structural decision** that prevents the burnout of endless back-to-back delivery cycles and gives the organisation breathing room to think before committing to the next cycle.

---

#### The Bet Metaphor

Singer uses betting deliberately. When you bet on a pitch:

- You are committing real resources — six weeks of a team's time
- You accept the risk that it might not work out
- You are **not** making a promise to deliver specific features by a specific date

This reframes the relationship between leadership and teams: leadership makes bets, teams honour them by working autonomously toward the best outcome. It is not a contract for specific deliverables — it is a commitment of time to solve a defined problem.

---

### Part 3: Building

---

#### Small Autonomous Teams

A typical Shape Up team is **one designer and one or two developers**. Sometimes just two people total. This is not a resource constraint — it is a deliberate design decision.

Small teams:
- Have lower coordination overhead
- Make decisions faster
- Maintain shared context without meetings
- Feel genuine ownership of the outcome

The team is given the shaped pitch and then **left alone** to figure out how to execute it. No daily standups from management. No mid-cycle scope additions. No interruptions.

---

#### No Handoffs

In traditional processes, work passes through hands: product manager writes spec → designer creates mockups → developer implements → QA tests → product manager reviews. Each handoff loses context, introduces delay, and diffuses ownership.

In Shape Up, **the team does everything together**. Designer and developer work side by side from day one. Design decisions are made with full awareness of implementation constraints. Implementation decisions are made with full awareness of design intent.

---

#### Scope Hammering

One of the most practical concepts in the book. When a team discovers mid-cycle that the work is bigger than expected, the answer is never to extend the deadline. The answer is to **hammer the scope**.

Singer introduces the concept of work as an **iceberg** — what's visible above the water is the core value, what's below is the nice-to-haves, the edge cases, the polish. When time runs short, you cut below the waterline — but you protect above it.

This requires distinguishing:
- **Must-haves** — the core of what makes the feature valuable
- **Nice-to-haves** — things that would be good but don't define success
- **No-gos** — explicitly out of scope

Scope hammering is not failure. It is **professional judgment** about what matters most. The team that ships a complete core feature is more successful than the team that ships 80% of an over-scoped one.

---

#### Hill Charts — Tracking Progress Honestly

Traditional progress tracking asks: *"What percentage is done?"* This produces dishonest estimates because the unknown unknowns are invisible.

Singer introduces the **hill chart** — a simple visual metaphor:

```
        ▲
       /|\
      / | \
     /  |  \
    /   |   \
---/----+----\---
 Figuring    Execution
 it out      (known)
(unknown)
```

Every piece of work starts on the uphill side — the team is still figuring out the approach. Once they've crested the hill — once they know exactly what they're building and how — work moves to the downhill side and progress becomes predictable.

The insight: **uphill work is fundamentally different from downhill work**. Uphill work involves discovery, uncertainty, and potential pivot. Downhill work is execution. Treating them the same way produces dishonest progress reports.

Hill charts let teams and leadership see at a glance which pieces of work are still in discovery mode — where surprises might still come — versus which are in execution mode, where delivery is reliable.

---

#### Circuit Breaker

If a team reaches the end of a six-week cycle and the work isn't done, the default is **not to extend**. The project is stopped.

This sounds harsh but is deliberate. It:
- Forces scope hammering during the cycle rather than at the end
- Prevents bad projects from consuming unlimited time
- Surfaces problems early — if a project consistently can't ship in six weeks, the shaping was wrong
- Respects the betting table's role — extending a project is effectively making a new bet, which should be a conscious decision

If the work is genuinely worth continuing, it gets **reshaped and re-pitched** for a future cycle. The circuit breaker ensures that extension is always a deliberate choice, not an automatic assumption.

---

### The Underlying Philosophy

Shape Up is built on a set of beliefs about software development that run counter to most industry practice:

**Time is a design tool, not just a constraint**
Setting the appetite before shaping the solution produces better, simpler solutions than estimating after design.

**Autonomy requires clarity at the boundary**
Teams can only be truly autonomous if what they're responsible for is crystal clear. Shaping provides that clarity without over-specifying the solution.

**Backlogs are a form of debt**
The discipline of no backlog forces genuine prioritisation and prevents the accumulation of stale commitments.

**Fixed time, variable scope**
The opposite of the traditional model. Time is the constant; scope is what flexes. This produces honest delivery over delusional promises.

**Done means shipped**
A feature that's 95% complete and not in production is not done. Shape Up optimises relentlessly for real, shipped, usable software.

---

### The Central Message

> **Most teams aren't slow because their developers are slow. They're slow because the work arriving at developers is the wrong shape — too vague to start, too prescriptive to own, too large to finish, and too disconnected from real value to prioritise honestly. Fix the shape of the work before it reaches the team, give the team real autonomy and a real deadline, and then get out of the way. Shipping something real in six weeks beats planning something perfect for six months.**

Shape Up is not a process to follow. It is a philosophy of **respect** — respect for the team's autonomy, respect for the organisation's time, and respect for the user's actual need over the product manager's imagined solution.
