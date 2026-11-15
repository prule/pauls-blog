---
title: "Six Builds of One API: What the math-api Experiments Showed"
date: 2026-11-15T10:00:00+11:00
publishDate: 2026-11-15
draft: false
layout: "post"
tags: ["claude-code", "openspec", "ponytail", "code-review", "multi-agent", "sdlc-pipeline", "ai-assisted-coding", "process"]
categories: ["coding"]
description: "A summary of six builds of the same Spring Boot API with Claude Code, comparing a five-agent pipeline, OpenSpec and plain prompting: what each cost, what each caught, and what I'd use by default."
author: "Paul"
---

Use one agent with reviews from the first change, and write the rules down exactly. That's the default the math-api experiments point to after six builds of the same API.

- **The process didn't change the design.** Every build ended up with the same layers, the same `Operation` enum and the same Jackson limits. The docs decided the architecture, not the workflow.
- **Reviews are cheap.** A plan review and two code reviews per change added about **8%** in time and tokens. They found more issues than a five-agent pipeline that cost **2.4× the time**.
- **OpenSpec's overhead depends on the backlog.** It cost **52% more time** when the backlog already held the design, and **16% more** when the builder had to design. Either way it wrote about **1.4× the output tokens**.
- **Written rules catch bugs.** The same `Math.abs(scale)` bug appeared in five builds. Each build that caught it had the rule written down as two comparisons, or had tests that were proven to fail.

This post sums up the four detailed write-ups. The numbers, probes and caveats are in those posts.

## The builds

Each build used the same PRD, domain model and standards, ran on Claude Opus 5.5, and stopped at item 5 (OpenAPI docs).

| Build | Process | Active time, items 2–5 | Tests | `Math.abs(scale)` bug |
|---|---|---|---|---|
| [math-api](https://github.com/prule/math-api) | One agent, OpenSpec, no reviews | 47 min | 108 | Avoided (cast to `long`) |
| [math-api-sdlc-pipeline](https://github.com/prule/math-api-sdlc-pipeline) | Five agents, two human gates | 112 min | 155 | Caught by QA mutation |
| [math-api-one-agent-2](https://github.com/prule/math-api-one-agent-2) | One agent, OpenSpec, plan and code reviews | 50 min | 129 | Caught by code review, against design.md |
| [math-api-vibe-coded](https://github.com/prule/math-api-vibe-coded) | One agent, no OpenSpec, code reviews | 33 min | 73 | **Shipped**: returned a 500 |
| [math-api-outcome-based-backlog-openspec](https://github.com/prule/math-api-outcome-based-backlog-openspec) | As one-agent-2, outcome backlog, no plan review | 46 min | 127 | Caught by the agent, against design.md |
| [math-api-outcome-based-backlog-vibe](https://github.com/prule/math-api-outcome-based-backlog-vibe) | As vibe-coded, outcome backlog | 40 min | 102 | **Shipped**: latent, as Jackson rejects the input first |

The first four builds used an implementation-level backlog, where each prompt named the classes, status codes and tests. The last two used an outcome backlog, such as "clients can divide", with "done when" checks.

{{< mermaid >}}
flowchart LR
  A["math-api<br/>one agent"] --> B["sdlc-pipeline<br/>five agents"]
  B --> C["one-agent-2<br/>+ reviews"]
  C --> D["vibe-coded<br/>− OpenSpec"]
  D --> E["outcome backlog<br/>vibe vs openspec"]
{{< /mermaid >}}

Each experiment came from the caveat of the one before it.

## 1. The docs decided the design, not the process

Put any two of the six codebases side by side and they're near twins: `domain / application / adapter.in.web / config` enforced by ArchUnit, BR-1 checked in the `Calculation` constructor, one Problem Details advice, and the same three Jackson read limits, found separately each time. Production code stayed between about 430 and 560 lines.

Five agents and two gates didn't change the architecture. Neither did dropping OpenSpec. A clear PRD, domain model and standards had already fixed it. Process changed what got *verified*, not what got built.

## 2. Reviews give most of the pipeline's value, for a fraction of the cost

[One Agent vs Five]( {{< ref "post/coding/2026/one-agent-vs-five/" >}} ) found the pipeline cost about 2× the time and 2.6× the output tokens. Most of that went to cold context and planning: every agent re-reads the docs and code from nothing.

[Reviews Without the Pipeline]( {{< ref "post/coding/2026/reviews-without-the-pipeline/" >}} ) added reviews to the single agent instead:

- a ponytail review of each plan;
- a ponytail review of each diff;
- `/code-review`, with two subagents checking against the standards and the spec.

That cost **about 8% more** than no reviews, recorded **45 findings against the pipeline's 36**, and ran at **about 40% of the pipeline's cost**. The two code reviews overlapped on only 3 of 45 findings. Ponytail found wrong code. `/code-review` found drift from what the repo had written down.

The pipeline kept one clear advantage: **QA by mutation**. It broke each change 8–21 times on purpose to prove the tests could fail. No other build did that.

## 3. OpenSpec's cost is mostly writing, and it pays for exact rules

| Items 2–5 | vibe | openspec | time | tokens |
|---|---|---|---|---|
| Task backlog (design in the prompt) | 32.6 min | 49.6 min | 1.52× | 1.39× |
| Outcome backlog (builder designs) | 40.1 min | 46.4 min | 1.16× | 1.43× |

With a task backlog, OpenSpec's propose step mostly restated a design the prompt already gave ([Part 1]( {{< ref "post/coding/2026/vibe-coding-vs-openspec/" >}} )). With an outcome backlog, the vibe build had to design too, so it slowed by 23% and the time gap mostly closed ([Part 2]( {{< ref "post/coding/2026/outcome-backlog-vibe-vs-openspec/" >}} )). The token gap didn't close: proposal, design, spec delta and task files are about 40% more text whatever the backlog.

What OpenSpec bought:

- **design.md as a target for reviewers.** It gave reviewers and the agent an exact statement to check the code against. That's how the BR-1 bug was caught, twice.
- **Branch coverage, sometimes.** Spec scenarios became tests in Part 1 (94% against 80%). In Part 2 the result reversed (82.5% against 95%), because OpenSpec added defensive code that no test reached.

What it didn't buy:

- **Living specs.** They mostly restated the PRD and domain model. No change broke old behaviour, so the specs never got to prove their main value.
- **A plan checked before code.** Without a plan review, 11 scenarios were added to the plan after code review. The plan then recorded what the review had decided.
- **`/opsx:verify`.** Of its 8 findings, 5 were false alarms about an unticked CI task.

## 4. One bug, five builds

BR-1 caps an Operand's scale at ±1000. Five of the six builds wrote this check at some point:

```java
if (Math.abs(operand.scale()) > MAX_SCALE) { ... }
```

`Math.abs(Integer.MIN_VALUE)` is still negative, so `1E+2147483648` gets past the check. The original avoided the bug by casting to `long`.

Each build that caught the bug had an independent statement of intent to compare the code against:

- **pipeline:** a mutant proved that a test couldn't fail.
- **one-agent-2:** both code reviews compared the code with design.md, which said "two comparisons".
- **outcome openspec:** design.md said `-1000 <= scale() <= 1000`, and the agent noticed its own code didn't match.

Both vibe builds had no such statement, and all their reviewers read the line and moved on. The model made the same mistake every time. What changed the outcome was whether something written down disagreed with the code.

{{< notice type="info" >}}
This is one data point per build, not a rate. But it points the same way five times: the written rule is the part of the plan that matters, not the ceremony around it.
{{< /notice >}}

## 5. Patterns that held across every build

- **Reviewers that run things find real bugs.** The best catches came from agents running the claim, not reading it: Jackson probes, `jshell`, mutants, the spec reviewer sending requests to the running jar. Reading found drift. Running found bugs.
- **Retrospectives repeat, they don't compound.** "Check the framework's defaults before planning" came up in four of five changes in one-agent-2. "Probe the running app with hostile input" was relearned in item 2 of both outcome builds. A lesson only sticks once it's in the file the next step reads: the propose prompt, STANDARDS, CLAUDE.md.
- **Process wording matters more than tooling.** The vibe session stopped twice to wait for a nudge. The OpenSpec session didn't, because its CLAUDE.md said "run each step yourself". That's one sentence, not a framework.
- **Docs carry decisions forward.** Each later build started from docs that held the earlier builds' decisions. That made every comparison less clean, and it shows where the value lives.

## What I'd use by default

{{< mermaid >}}
flowchart LR
  I[Backlog item] --> R{Risky?<br/>new rule, boundary,<br/>parser, security}
  R -- no --> F["One agent<br/>+ ponytail + /code-review"]
  R -- yes --> G["Write exact rules first<br/>(design.md or PR 'Design' section)<br/>+ mutation pass in review"]
{{< /mermaid >}}

- **One agent, with both code reviews, from item 1.**
- **Write the rules exactly as they'll be coded** before the code, for any change with a boundary or a limit. That can be design.md or a "Design" section in the PR. The reviewer checks the code against it.
- **Add a mutation pass to code review.** It's the pipeline's one unique win, and it's cheap to run on a scratch copy.
- **Make probing the running app a process step**, including numeric guards at their type limits.
- **Write recurring lessons into the files they target**, not into retrospectives.
- **Keep the full pipeline** for changes where a separate plan and QA pass is worth roughly twice the time.

## Next: code as the only spec

A seventh repo, [math-api-kotlin-migration](https://github.com/prule/math-api-kotlin-migration), ports one-agent-2 from Spring Boot to Kotlin and Ktor. The rule: the Java code, tests and config are the only spec, and the docs are off-limits. A black-box contract suite records golden responses from the Java service first. The Kotlin service must then match them on the wire, apart from a logged list of accepted differences. Health, calculations and OpenAPI are ported, CI proves equivalence on every change, and the v2 audit gate is in. That gets its own post.

## Caveats

- **One run per build.** Single results, such as which reviewer caught Jackson's coercion, came out differently across rounds with the same brief.
- **The builds weren't independent.** Each started from docs improved by the builds before it, using the same model.
- **Findings were counted differently.** Pipeline gates, retrospective bullets and review tallies don't share a unit.
- **Time is an estimate.** Active time sums gaps between log events across the main session and subagents, and skips gaps over 3 minutes.
