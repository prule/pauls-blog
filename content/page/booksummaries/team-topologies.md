---
title: "Team Topologies"
description: "A summary of Team Topologies by Matthew Skelton and Manuel Pais, focusing on team design as a first-class engineering decision."
layout:     page
draft: false
---


## Team Topologies — Summary

**By Matthew Skelton & Manuel Pais**

{{< notice type="info" title="Amazon Affiliates Link" >}}
Buy on Amazon [Team Topologies](https://amzn.to/3QR8qg9)
{{< /notice >}}

---

### Core Thesis

Most organisations design their teams around org charts, project needs, or historical accident — and then wonder why their software architecture is a mess and their delivery is slow. Team Topologies argues that **team design is a first-class engineering decision**, as important as any technical choice you make.

The book's central claim: **your team structure will become your system architecture** whether you intend it or not. So you might as well design both deliberately.

> This is Conway's Law stated as a design tool rather than a warning: *"Any organisation that designs a system will produce a design whose structure is a mirror image of the organisation's communication structure."*

---

### The Cognitive Load Problem

Before introducing team types, Skelton and Pais establish the foundational constraint that everything else flows from: **cognitive load**.

Every team has a finite capacity for complexity. When the scope of a team's responsibility exceeds that capacity, delivery slows, quality drops, context switching increases, and people burn out. Yet most organisations keep piling more domains, more services, and more responsibilities onto existing teams without ever asking whether the cognitive load is sustainable.

**The primary design goal of Team Topologies is to minimise and manage cognitive load across the organisation.**

Three types of cognitive load matter:
- **Intrinsic** — the inherent complexity of the domain itself
- **Extraneous** — complexity imposed by tools, processes, and poor organisation
- **Germane** — the useful cognitive work of learning and building expertise

Good team design eliminates extraneous load and protects space for germane load.

---

### The Four Team Topologies

---

#### 1. Stream-Aligned Team
The primary team type. Aligned to a **flow of work** — a product, a user journey, a business domain. This team owns a slice of the system end-to-end and can deliver value independently without waiting for other teams.

**Characteristics:**
- Small, long-lived, stable membership
- Full ownership of their domain — build, run, support
- Minimal handoffs to other teams
- Empowered to make most decisions within their stream

**The goal:** A stream-aligned team should be able to deliver continuously without being blocked by dependencies on others. If they're constantly waiting, the topology is wrong.

This is the team type that **all other team types exist to serve**.

---

#### 2. Enabling Team
A small group of specialists whose job is to **grow the capabilities of stream-aligned teams** — not to do the work for them.

**Characteristics:**
- Deep expertise in a specific area (security, observability, testing, UX)
- Temporary engagement model — they work with a team, upskill them, then move on
- Explicitly *not* a dependency — the goal is to make themselves unnecessary
- Act as consultants and coaches, not gatekeepers

**The failure mode:** An enabling team that becomes a permanent bottleneck — every team must go through them to do anything in their domain. This turns a capability-builder into a blocker.

---

#### 3. Complicated Subsystem Team
Owns a **technically complex component** that requires deep specialist knowledge — a machine learning pipeline, a real-time physics engine, a custom compression algorithm.

**Characteristics:**
- Exists because the complexity would overwhelm a stream-aligned team
- Provides a well-defined interface to other teams
- The interface is as important as the subsystem itself — it must be stable and clear
- Should be as small as possible — only created when complexity genuinely demands it

**The trap:** Creating complicated subsystem teams for things that aren't truly complicated — this just adds coordination overhead and reduces stream-aligned team autonomy.

---

#### 4. Platform Team
Provides **internal services that reduce the cognitive load of stream-aligned teams** by handling common infrastructure, tooling, and operational concerns.

**Characteristics:**
- Treats stream-aligned teams as customers
- Builds self-service capabilities — APIs, pipelines, deployment tools, observability dashboards
- The test of a good platform: stream-aligned teams can use it without needing to talk to the platform team
- Should make the easy path the right path

**The critical mindset shift:** A platform team that requires constant interaction, tickets, and coordination has failed. The platform should be **a product**, not a service desk.

> "The platform team's job is to make doing the right thing easy and doing the wrong thing hard — invisibly."

---

### The Three Interaction Modes

How teams work together matters as much as how teams are structured. Skelton and Pais define three — and only three — interaction modes.

---

#### 1. Collaboration
Two teams work **closely together** for a defined period to discover something neither could discover alone — new technology, new domain understanding, a new architectural approach.

- High bandwidth, high overhead
- Appropriate for exploration and uncertainty
- **Should be temporary** — collaboration that continues indefinitely becomes permanent dependency

---

#### 2. X-as-a-Service
One team **consumes something** provided by another team with minimal interaction — an API, a platform capability, a data pipeline.

- Low bandwidth, low overhead
- Appropriate for stable, well-understood interfaces
- Requires the providing team to invest heavily in documentation, reliability, and self-service

---

#### 3. Facilitating
One team **helps another team** learn or improve — coaching, consulting, temporarily embedding.

- The enabling team interaction mode
- Goal is knowledge transfer, not dependency creation
- Should have a clear end point

---

**Why only three?**

Because ad hoc, undefined interaction modes are where cognitive load goes to compound. When teams interact in unstructured ways — constant Slack messages, informal favours, tribal knowledge dependencies — the organisation accumulates invisible coordination debt that slows everything down.

Naming and limiting interaction modes forces intentionality.

---

### Conway's Law as a Design Tool

Most engineers know Conway's Law as a cautionary observation. Team Topologies reframes it as an **architectural lever**.

**Inverse Conway Manoeuvre:** Design your team structure to match the software architecture you *want* to produce — not the one you currently have.

If you want loosely coupled microservices, create loosely coupled teams with clear boundaries. If your teams are tightly coupled — sharing codebases, requiring constant coordination, unable to deploy independently — your architecture will be tightly coupled regardless of what the architecture diagram says.

The implication: **you cannot fix your architecture without fixing your team structure**, and vice versa.

---

### Team-First Thinking

Several principles flow from putting the team — not the individual, not the project — at the centre:

**Dunbar's Number applied to teams:**
- Maximum ~15 people for high trust within a team
- Maximum ~50 people for a tribe or group
- Maximum ~150 for an organisation where everyone knows everyone
  These are not arbitrary — they reflect the cognitive limits of human relationship maintenance.

**Stable teams over project teams:**
Project teams form, learn to work together, deliver, then disband — wasting all the accumulated context and trust. Long-lived stable teams that *own* a domain continuously outperform temporary project teams dramatically over time.

**The team owns the work — not the individual:**
Assigning work to individuals rather than teams creates knowledge silos, bus factor risk, and hero culture. Assigning work to teams creates shared ownership, collective knowledge, and resilience.

---

### The Shape of a Healthy Organisation

A well-designed organisation under Team Topologies looks something like:

- Several **stream-aligned teams** — each owning a domain end-to-end, deploying independently, minimally blocked
- One or more **platform teams** — providing self-service infrastructure that stream-aligned teams consume without friction
- Small **enabling teams** — moving around the organisation, growing capabilities, making themselves redundant
- Occasional **complicated subsystem teams** — for the genuinely hard bits that would otherwise overwhelm stream teams

The ratios matter: stream-aligned teams should be the **majority**. Every other team type should be justified by the cognitive load relief it provides to stream-aligned teams.

---

### Connecting Back to Software Architecture

The book's most powerful practical tool: **use your desired architecture to draw your desired team structure, then work backwards to get there**.

| Desired Architecture | Implied Team Structure |
|---|---|
| Independent deployable services | Stream-aligned teams with clear domain ownership |
| Shared platform capabilities | Platform team with self-service interface |
| Complex ML / data pipeline | Complicated subsystem team with clean API |
| Organisation-wide security posture | Enabling team that upskills, not a gatekeeper |
| Monolith with poor modularity | Teams too coupled — restructure both together |

---

### Adding to the Synthesised Philosophy

Team Topologies adds a dimension that the previous books only touched on — **the organisational and social architecture** that surrounds the code:

| Principle | Source |
|---|---|
| Team structure becomes system structure — design both deliberately | Team Topologies |
| Cognitive load is a finite resource — protect it ruthlessly | Team Topologies |
| Long-lived stable teams outperform project teams | Team Topologies |
| Platform teams serve stream teams — not the other way around | Team Topologies |
| Interaction modes should be named, limited, and intentional | Team Topologies |
| Use Conway's Law as a design tool, not just a warning | Team Topologies |

---

### The Central Message

> **Software architecture and team architecture are the same problem viewed from different angles. You cannot fix one without addressing the other. Design your teams around cognitive load, flow of value, and the system you want to build — and the system will follow. Leave team design to chance, and your architecture will reflect that chaos faithfully.**

It is the missing layer between individual engineering practice and organisational delivery — the bridge between how one person writes code and how hundreds of people ship a coherent system together.

---

---

## Remote Team Interactions Workbook — Summary

**By Matthew Skelton & Manuel Pais — Companion to Team Topologies**

{{< notice type="info" title="Amazon Affiliates Link" >}}
Buy on Amazon [Remote Team Interactions Workbook: Using Team Topologies Patterns for Remote Working](https://amzn.to/4egbsEG)
{{< /notice >}}

---

### Core Purpose

This workbook is a **practical field guide** for applying Team Topologies in remote and hybrid environments. Where Team Topologies established the theory — four team types, three interaction modes, cognitive load as the primary constraint — this workbook asks: *how do you actually make that work when your teams are distributed across cities, time zones, and kitchen tables?*

The central argument: **remote work doesn't break Team Topologies — but it does make every weakness in your team design immediately visible.** The ambiguities and informal coordination that limp along in an office collapse entirely when teams are distributed.

> **"Remote work is a stress test for your organisation. Whatever was unclear becomes broken. Whatever was clear becomes your competitive advantage."**

---

### The Remote Amplification Effect

The workbook opens with a critical observation: remote work **amplifies existing organisational dynamics** — good and bad.

- Teams with clear ownership and clean interfaces continue to deliver well, often better — fewer interruptions, more focus
- Teams with unclear boundaries, high coupling, and informal coordination **grind to a halt** — the hallway conversations that papered over structural problems no longer exist

This means the first step in improving remote team performance is almost never about tools or processes. It is about **getting the underlying team topology right**.

---

### Rethinking the Four Team Types Remotely

---

#### Stream-Aligned Teams — Remote Considerations

The good news: stream-aligned teams are the **most naturally suited** to remote work, because their defining characteristic — end-to-end ownership of a domain with minimal dependencies — means they don't need constant cross-team interaction.

The risks remotely:
- **Invisible overload** — when a team member is struggling with cognitive load, it's much harder to notice remotely than in person
- **Domain drift** — without physical proximity cues, the boundaries of what a team owns can blur over time
- **Onboarding collapse** — bringing new members into a stream-aligned team is dramatically harder remotely; tacit knowledge that transfers naturally in an office requires deliberate documentation and structured pairing

**Practical responses:**
- Make team boundaries and ownership explicit in writing — not just understood
- Build regular explicit check-ins on cognitive load, not just delivery status
- Treat onboarding as a first-class engineering problem with the same rigour as any other

---

#### Enabling Teams — Remote Considerations

Enabling teams face a particular challenge remotely: their work is inherently **relational**. Coaching, upskilling, and building capability in other teams requires trust and rapport — which is harder to build across a video call.

The failure modes:
- Enabling teams becoming **asynchronous ticket queues** rather than genuine coaches
- Stream-aligned teams not knowing the enabling team exists or what they offer
- Engagements ending without real capability transfer because the relationship was too shallow

**Practical responses:**
- Deliberately invest in relationship-building at the start of every enabling engagement — not just task handoff
- Use structured pairing and mob sessions rather than documentation drops
- Define explicit capability milestones so both teams know when the engagement is genuinely complete
- Make enabling team availability and current engagements visible organisation-wide

---

#### Platform Teams — Remote Considerations

Platform teams are arguably **better suited to remote work** than any other type, because their output — self-service tools, APIs, pipelines — is inherently digital and asynchronous. A good platform doesn't require a conversation.

The risks:
- Platform teams defaulting to **synchronous support** because async feels less helpful — this defeats the purpose
- Documentation quality becoming the primary differentiator between a good and bad platform remotely
- Platform teams losing awareness of friction experienced by stream-aligned teams — the casual "this is really annoying" conversation that happens in an office doesn't happen over Slack

**Practical responses:**
- Treat documentation as a product, not an afterthought — it is the primary interface remotely
- Build explicit feedback channels from stream-aligned teams back to the platform team
- Run regular developer experience reviews — structured sessions where stream teams walk through their platform interactions and surface friction points
- Resist the pull toward becoming a service desk — every ticket that could be self-service is a platform improvement opportunity

---

#### Complicated Subsystem Teams — Remote Considerations

These teams are often composed of deep specialists who may already work somewhat independently. Remote work can suit them well — but creates risks around:

- **Interface drift** — without regular interaction, the interface a subsystem team provides can diverge from what stream-aligned teams actually need
- **Isolation** — specialist teams can become disconnected from organisational context, building the technically perfect thing that solves yesterday's problem

**Practical responses:**
- Regular interface reviews with consuming teams — not just when something breaks
- Explicit roadmap sharing so subsystem evolution stays aligned with stream-aligned team needs

---

### The Three Interaction Modes — Remote Realities

This is where the workbook adds the most practical value. Each interaction mode requires deliberate redesign for remote contexts.

---

#### Collaboration Mode — Remotely

Collaboration is the **hardest interaction mode to run remotely**. It requires high bandwidth, rapid iteration, and the ability to build shared understanding quickly. All of these are harder without physical proximity.

**The remote collaboration toolkit:**
- **Synchronous sessions with shared digital workspace** — Miro, FigJam, or equivalent. Not just a video call with one person sharing a screen
- **Explicit collaboration contracts** — what are we trying to discover? What does success look like? When does this collaboration end?
- **Shorter, more frequent sessions** over fewer, longer ones — remote collaboration fatigue is real
- **Documented decisions in real time** — the shared understanding built in a room dissipates fast remotely; capture it as it forms

**The time zone problem:**
True collaboration requires significant overlap. Teams more than 3-4 time zones apart cannot sustain genuine collaboration mode without someone working outside normal hours — which is unsustainable. The workbook is honest about this: **sometimes the answer is that collaboration mode is not available across a given time zone gap**, and the team structure needs to reflect that.

---

#### X-as-a-Service Mode — Remotely

This is the **most naturally remote-compatible** interaction mode. If a team is consuming a well-defined service asynchronously, geography is largely irrelevant.

The key remote requirements:
- **Exceptionally clear documentation** — there is no "quick question to the team next door"
- **Stable, versioned interfaces** — breaking changes need more lead time and more explicit communication remotely
- **Async-first support model** — response SLAs, not real-time availability

**The hidden risk:** X-as-a-Service mode can mask a relationship that should actually be collaboration mode. If a consuming team is constantly asking questions, seeking clarification, and working around undocumented edge cases — that's not X-as-a-Service working. That's collaboration mode that hasn't been acknowledged.

---

#### Facilitating Mode — Remotely

Facilitation remotely requires more **structure and intentionality** than in person. The informal coaching that happens naturally in a shared office — overhearing a problem, dropping by to help — must be replaced with deliberate mechanisms.

**Practical approaches:**
- Scheduled pairing sessions rather than ad hoc help
- Structured retrospectives on capability growth, not just delivery
- Office hours — predictable async-friendly windows where the enabling team is available
- Written learning resources created during the engagement, not after

---

### Team Interaction Mapping — The Workbook's Core Tool

The workbook's primary practical contribution is a **structured process for mapping your current team interactions** and identifying where they are misaligned with the desired topology.

**The mapping process:**

**Step 1 — Identify your teams**
List every team. Assign each to one of the four types — be honest about what they actually are, not what they're supposed to be.

**Step 2 — Map current interactions**
For every pair of teams that interact regularly, classify the interaction as collaboration, X-as-a-Service, or facilitating. If you can't classify it, that's a signal — undefined interaction modes are organisational debt.

**Step 3 — Identify mismatches**
Common mismatches:
- A platform team running as a collaboration team — too much back and forth
- An enabling team that has become a permanent dependency — never graduating the teams it works with
- A stream-aligned team running in collaboration mode with five other teams simultaneously — cognitive load crisis

**Step 4 — Design the desired state**
What interaction mode *should* each team pair be using? What would need to change structurally to get there?

**Step 5 — Identify the evolution path**
How do you move from current state to desired state? What changes to team ownership, interfaces, documentation, and tooling are required?

---

### Remote-Specific Failure Modes

The workbook catalogues patterns that appear specifically in remote organisations:

---

**The Zoom Sprawl**
Every undefined interaction becomes a meeting. Teams without clear interaction modes default to synchronous calls for everything — which destroys focus, creates timezone inequity, and scales terribly.

*Fix:* Define which interactions are async-first by default. Meetings should be the exception for coordination, not the mechanism.

---

**The Silent Overload**
Remote team members hit cognitive load limits without anyone noticing. In an office, visible stress, late nights, and body language are signals. Remotely, people suffer quietly until they leave.

*Fix:* Explicit, regular cognitive load check-ins built into team rhythm — not just delivery check-ins. "How much is on your plate?" asked deliberately, not assumed from ticket counts.

---

**The Documentation Desert**
Teams that relied on tribal knowledge and hallway conversation have nothing written down. Remotely, this means new members can't onboard, decisions can't be traced, and interfaces can't be consumed without human support.

*Fix:* Documentation is infrastructure. Treat it with the same rigour as code — reviewed, maintained, versioned.

---

**The Async Black Hole**
Messages sent into Slack or email and never actioned. Decisions that need to be made sitting in threads for days. Stream-aligned teams blocked on responses from platform or enabling teams.

*Fix:* Explicit response SLAs for different interaction types. Not "we'll get back to you" but "X-as-a-Service requests get a response within 24 hours."

---

**The Timezone Fault Line**
Organisations that nominally operate globally but actually have a dominant timezone — usually where leadership sits — that makes decisions synchronously while other timezones are asleep.

*Fix:* Audit which decisions require synchronous input and which don't. For genuinely global teams, rotate meeting times, document decisions asynchronously, and give distributed team members real — not just nominal — decision-making authority.

---

**The Collaboration Collapse**
Two teams nominally in collaboration mode but in different timezones with only two hours of overlap. The collaboration produces nothing because there isn't enough synchronous bandwidth to build shared understanding.

*Fix:* Either co-locate the collaboration temporarily, restructure to avoid needing collaboration across that time gap, or accept that the topology needs to change.

---

### Sensing Mechanisms — How to Know When Something Is Wrong

One of the workbook's most practical contributions: a set of **signals that your team topology is misaligned**, adapted for remote environments where casual observation isn't available.

| Signal | What It Likely Means |
|---|---|
| Team constantly waiting on another team | Wrong interaction mode — or ownership boundary is wrong |
| Platform team overwhelmed with support tickets | Platform isn't self-service enough |
| Enabling team engagement never ends | Not transferring capability — creating dependency |
| New team members take months to become effective | Onboarding is broken — documentation deficit |
| Decisions take weeks that should take days | Too many teams in the decision chain |
| Same questions asked repeatedly across teams | Knowledge isn't being captured or shared |
| High performer leaves and delivery collapses | Single point of knowledge — not a team, a hero |

---

### Connecting to the Broader Philosophy

Remote Team Interactions Workbook adds the final practical layer to the philosophy we've been building:

| Principle | Source |
|---|---|
| Remote work amplifies existing topology problems — fix the topology first | Remote TI Workbook |
| Define and name every interaction mode explicitly — ambiguity costs more remotely | Remote TI Workbook |
| Documentation is infrastructure, not afterthought | Remote TI Workbook |
| Cognitive load monitoring must be deliberate when you can't see people | Remote TI Workbook |
| Async-first by default — synchronous by exception and intention | Remote TI Workbook |
| Time zone gaps constrain which interaction modes are available | Remote TI Workbook |
| Map current interactions before designing future ones | Remote TI Workbook |

---

### The Central Message

> **Remote work does not change what good team design looks like — it simply removes every excuse for not doing it properly. The informal coordination, the hallway conversations, the ambient awareness of a shared office — these were always workarounds for structural problems. Distributed teams must replace them with deliberate design: explicit ownership, named interaction modes, documented interfaces, and genuine self-service platforms. Get the topology right, and remote works. Leave it to chance, and distance will finish what ambiguity started.**
