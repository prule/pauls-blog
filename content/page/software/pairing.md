---
title: "Collaborative Development"
layout:     page
draft: false
---

## Collaborative Development — Beyond Strict Pair Programming

*(updated to include mentoring and teaching)*

---

### The Spectrum of Collaboration

Traditional pair programming has a specific form: two people, one keyboard, one screen, switching between driver and navigator roles. The research behind it is solid — it produces fewer defects, spreads knowledge, and keeps both people focused. But the strict form has real costs: it's cognitively intense, it doesn't suit all personality types, and it treats every problem as requiring the same level of collaboration.

The more useful idea is that **collaboration exists on a spectrum**, and the right point on that spectrum depends on the problem, the people, and the moment.

---

### What Makes Collaboration Valuable

Before choosing a form, it's worth naming what you're actually after:

- **A second perspective** — someone to notice what you've stopped seeing
- **Faster problem decomposition** — two people mapping a problem space find the edges quicker
- **Decision confidence** — a sounding board before committing to an approach
- **Knowledge transfer** — understanding spreading naturally through working together rather than through documentation or formal review
- **Momentum** — being stuck alone is demoralising; being stuck together is a problem to solve

Not all of these require constant presence or a strict protocol to achieve.

---

### Modes of Working Together

**Problem framing together, execution separately**

Spend time together at the start — whiteboard the problem, agree on the approach, identify the unknowns and risks. Then split off to implement independently. Reconvene when something unexpected surfaces or the work is ready for review. This suits problems that are well-understood once decomposed, and people who work better with uninterrupted focus time.

**Async collaboration with synchronous touchpoints**

One person works through a problem and leaves deliberate breadcrumbs — a draft PR, a design note, a short written summary of where they got to and what decision they're about to make. The other responds when context-switching is cheap rather than on demand. This respects deep work but keeps the second perspective in the loop. It works particularly well across different working rhythms or time zones.

**Side-by-side on hard problems only**

Most code doesn't need two people. Routine implementation, familiar patterns, mechanical refactoring — these are fine solo. But genuinely novel problems, architectural decisions, tricky bugs, and unfamiliar territory are where a second person pays dividends. Being deliberate about *when* to pull someone in — rather than pairing by default — means the collaboration is higher energy and higher value when it happens.

**One person drives the skeleton, the other fills it in**

A lightweight handoff pattern. One person sets up the structure — the test cases, the interface, the rough scaffold — and the other implements against it. This works naturally with TDD: writing the failing tests is a great collaborative act, because it forces both people to agree on behaviour before either writes implementation. The person who wrote the tests then reviews the implementation with full context.

**Walkthroughs and rubber ducking with teeth**

Not quite pair programming, but more than rubber ducking. One person explains what they're building or where they're stuck, and the other asks questions rather than giving answers. The questions do the work — "why does it need to know that?", "what happens if that's null?", "could that be a value object?". This draws on the same dynamic as pair programming's navigator role without requiring constant presence.

---

### Collaboration as Teaching

When experience levels differ, collaboration takes on a different character. The goal shifts — it's not just solving the problem at hand, but building the capacity of the less experienced person to solve the next one independently.

This changes how you show up as the more experienced person. The temptation is to reach for the keyboard and demonstrate — which is fast, but teaches very little. The more valuable approach is to stay in the questioning mode longer than feels comfortable: "what do you think the first step is?", "what would happen if we modelled it that way?", "what does the test tell you about the design?". Letting someone work through a wrong approach and discover why it's wrong themselves is often more durable than showing them the right one.

**Worked examples over finished solutions.** Walking through your own reasoning — including the dead ends — is more useful to a learner than a polished result. "I initially thought about doing it this way, but then I noticed this problem" transfers the thinking process, not just the answer. Finished solutions look inevitable in hindsight; the messy middle is where the actual learning is.

**Calibrating challenge.** Good teaching holds a person just beyond their current comfort zone — hard enough to stretch, not so hard they flounder without traction. In a coding context this means being honest about when to step in. If someone is genuinely stuck and not making progress, giving a nudge is more valuable than watching them spin. If they're making slow but real progress, staying quiet and letting them get there is usually the right call.

**Naming what's happening.** When a more experienced person makes a decision intuitively, it's worth narrating it explicitly — "I'm reaching for a value object here because I've been bitten by primitive obsession before" or "this is where I'd normally write the test first, because the interface isn't obvious to me yet." Tacit knowledge doesn't transfer unless it's made explicit. Things that feel obvious to a senior developer are often the hardest things for a junior to learn, precisely because nobody ever says them out loud.

**Letting ownership transfer gradually.** A common failure mode in mentoring is the more experienced person retaining de facto ownership — making the key decisions, doing the hard parts — while the less experienced person fills in the blanks. Genuine development requires the learner to hold the whole problem, make mistakes, and own the consequences. The mentor's job is to be a safety net, not a co-pilot.

---

### Teaching in Code Review

Code review is one of the highest-leverage mentoring tools available, because it's asynchronous, contextual, and leaves a written record. But its value as a teaching tool depends heavily on how it's done.

A review comment that says "use a value object here" closes the loop but doesn't build understanding. "What would happen if two different callers passed the arguments in the wrong order?" asks the same question but invites the person to reason through it. Over time, the goal is that the learner starts asking those questions themselves before submitting — internalising the reviewer's voice.

It's also worth distinguishing in review between things that matter and things that are personal preference. "This breaks the invariant because external code can now mutate the collection directly" is substantive. "I would have named this differently" is preference. Mixing the two trains people to treat all feedback as equally important — which means the important feedback gets lost.

---

### What to Agree on Upfront

Loose collaboration can drift into neither-person-is-really-responsible. A few lightweight agreements prevent this:

**Who owns the outcome.** Collaboration doesn't mean shared ownership by default. One person should be the accountable party — the one who makes the final call and carries it through. The other is a contributor.

**What kind of input is wanted.** "I want you to challenge this approach" is different from "I want a second pair of eyes on the implementation details." Knowing which is being asked for prevents mismatched collaboration — someone critiquing architecture when you wanted a typo check, or vice versa.

**When to reconvene.** Open-ended collaboration with no rhythm tends to collapse into one person working and the other occasionally checking in. Agreeing on specific sync points — at the start, when the interface is defined, before merging — gives structure without overhead.

---

### The Role of Code Review

In a collaborative-but-not-paired model, code review carries more weight. It becomes the primary venue for the second perspective to land — which means it needs to be substantive, not a rubber stamp.

Good review in this model goes beyond correctness. It asks whether the approach is right, whether the abstraction is well-named, whether the domain model reflects the business language, whether the tests describe behaviour clearly. This is closer to the navigator role in pair programming — high-level, questioning, oriented toward design — rather than a line-by-line error check.

The relationship between the reviewers matters here. Psychological safety — the confidence that raising a question won't be taken as a personal criticism — is what separates review that improves code from review that just delays it.

---

### The Underlying Principle

The value of a collaborator isn't constant presence — it's a different vantage point applied at the right moments. Someone who has just enough context to ask good questions, challenge assumptions, and notice what you've normalised is often more useful than someone who has been staring at the same problem for the same amount of time.

In a mentoring relationship, that principle extends further: the goal isn't just better code today, but a collaborator who needs less guidance tomorrow. The best mentoring makes itself gradually unnecessary.
