---
title: "One Agent vs Five: Building the Same API Twice"
date: 2026-11-10T12:00:00+11:00
publishDate: 2026-11-08
draft: false
layout: "post"
tags: ["claude-code", "openspec", "multi-agent", "sdlc-pipeline", "ai-assisted-coding", "process"]
categories: ["coding"]
description: "I built the same Spring Boot API twice, once with a single agent and once with a five-agent pipeline. The pipeline cost about 2× the time and 2.6× the tokens, gave the same design, and bought verification rather than better code."
author: "Paul"
---

A multi-agent pipeline doesn't produce a better design than a single agent. It produces a better-*verified* one, and that costs roughly **2× the time and 2.6× the output tokens**. Gate the risky changes. Fast-track the trivial ones.

That's the result of building [math-api](https://github.com/prule/math-api) twice from the same PRD, domain model, standards and backlog, up to the end of the MVP:

- **Original:** one agent working through [OpenSpec]( {{< ref "post/coding/2026/open-spec/" >}} ): propose, apply, archive, PR.
- **Rebuild:** [math-api-sdlc-pipeline](https://github.com/prule/math-api-sdlc-pipeline), using my `sdlc-pipeline` Claude Code plugin. Five specialist agents and two human gates per change.

| | Single agent | Pipeline |
|---|---|---|
| Active time, 5 changes | 75 min | 133 min |
| Output tokens | 217k | 562k |
| Cache writes | 368k | 2.68M |
| Review findings recorded | 0 | 36 |
| Tests | 108 | 155 |
| Production Java | 470 lines | 473 lines |

Four things follow from that table.

## 1. The extra cost is cold context and planning

Every agent starts with nothing. The architect, spec reviewer, implementer, QA and code reviewer each re-read the profile, the docs and the code. So the cost of loading context is paid five or six times per change instead of once. Cache writes went up **7×**. Output went up 2.6×. Cache reads only went up 1.4×, because the single agent's long session re-reads its own growing history on every turn anyway.

Where the pipeline's 85 agent-minutes went:

| Agent | Runs | Minutes | Share |
|---|---|---|---|
| architect | 12 | 32.9 | 39% |
| junior-dev | 6 | 20.3 | 24% |
| qa | 5 | 16.7 | 20% |
| spec-reviewer | 6 | 9.0 | 11% |
| senior-dev | 5 | 6.3 | 7% |

Planning is the largest share. The pipeline's input is a business-level use case, with no endpoints, classes or libraries, so the architect has to derive all of that. The original's backlog prompts already named classes and libraries. That's a design choice of the pipeline, and part of the price.

## 2. Same design, built independently

Put the two codebases side by side and they're near twins:

- `domain / application / adapter.in.web / config`, with the layering enforced by ArchUnit.
- An `Operation` enum with an exact, case-sensitive `named()` lookup.
- BR-1 operand limits checked in the `Calculation` record's constructor.
- One `ResponseEntityExceptionHandler` advice producing RFC 9457 Problem Details.
- **The same three Jackson read limits**, arrived at separately:

```yaml
spring.jackson.factory.constraints.read:
  max-document-length: 16384
  max-string-length: 1000
  max-number-length: 1000
```

Production code differs by three lines. Five agents and two gates didn't change the architecture; a good spec and good standards had already fixed it.

## 3. What you're buying is verification

The pipeline's value is evidence that the code does what the spec says:

- **Every scenario has a named test,** and QA breaks the code in a scratch copy to prove each test fails without its behaviour. That's 8–21 deliberately broken versions per change, and every gap it found was closed.
- **Plans are reviewed against the standards before any code is written.** Three findings on B-4 were wrong claims about what a value or library does, and each was caught by someone *running* the claim:
  - The spec reviewer ran Jackson itself and disproved the architect's risk example, before I was asked to accept that risk.
  - The plan labelled `98765E+1000` as out of range, but it sits exactly on the scale limit. The implementer stopped instead of weakening the test.
  - Two tests couldn't fail: one against a `Math.abs` scale check, one against echoing the value as `9.8765E+1005`. QA found both by breaking the code.
- **Every deviation from the plan is written down,** in design.md, tasks.md and a retrospective.

I ran both jars and sent them the same edge cases. Each wins on some:

| Case | Single agent | Pipeline |
|---|---|---|
| 405 / 415 framework errors | Problem Details **missing `type`**, which breaks the PRD | `about:blank`, with tests |
| OpenAPI examples | Hand-written JSON constants | Generated from the domain, each checked against a live response |
| Several required fields missing | Names every field | Names one |
| Operand out of range | Says which limit was broken | States both |
| Deeply nested 1.2 KB body | Generic "invalid request" | Wrongly says "too large" |

The `type` catch came from the pipeline's learning loop. B-1's retrospective said "list the framework defaults the design relies on". In B-2 that list made the implementer check Spring 7's Problem Details, and it found `type` left empty. The single agent shipped without it.

But the pipeline isn't strictly better. It lost on detail in three places, and its own audit found one of them.

## 4. Gates pay off on hard changes, not easy ones

Effort per change, in active minutes:

| Change | Single agent | Pipeline | Pipeline findings |
|---|---|---|---|
| Health check + walking skeleton | 30* | 22 | 4 |
| ADD / SUBTRACT / MULTIPLY | 12 | 25 | 9 |
| DIVIDE | 9 | 20 | 3 |
| Operand limits | 11 | 34 | 15, including 3 failed reviews |
| OpenAPI docs | 14 | 32 | 8 |

<small>* includes project setup.</small>

DIVIDE was one enum constant, one exception and one handler. The pipeline put it through all five agents and two gates, and found three wording nits. Operand limits, which involved parser internals, boundary values and failure ordering, is where the gates earned their keep.

So route by risk:

{{< mermaid >}}
flowchart LR
  UC[Use case] --> R{Risky?}
  R -- "new rule, boundary, parser, security" --> P[Full pipeline:<br/>architect → spec review → gate 1<br/>→ implement → QA → code review → gate 2]
  R -- "one constant, wording, config" --> F[Fast path:<br/>plan + implement + one review]
{{< /mermaid >}}

## The learning loop isn't closed yet

Each pipeline run writes a retrospective and updates a ledger of lessons. A lesson that recurs three times comes up at Gate 1 for me to adopt. After five changes there are 19 open lessons, and four have recurred twice.

But every lesson that "held" did so because the orchestrator pasted it into each agent's prompt. None has been written into its real target: STANDARDS, the project profile, `openspec/config.yaml`, or the agent definitions. Until then, the pipeline improves only as long as someone remembers to tell it. The fix is cheap: adopt the recurring lessons into those files.

{{< notice type="warning" >}}
**This comparison favours the pipeline.**

- It started from the original's *final* docs, so several hard decisions (failure order, limits, pinned versions) were already written down.
- I added ledger lessons to the agents' prompts by hand.
- The original had no review step until after v1, so its "0 findings" means *not looked for*, not *none*. By items 6–8 it ran a ponytail plan review and code review on every change, as in [this CLAUDE.md process]( {{< ref "post/coding/2026/claude-md-lightweight-process/" >}} ).
{{< /notice >}}

## What I'd change

- **Fast-track small changes.** Skip the separate spec-review round for a one-constant change.
- **Give the architect more context.** A technical note alongside the business use case would cut planning cost without leaking implementation into the use case.
- **Adopt recurring lessons into the files they target,** not the prompt.
- **Make the reviewers compute.** The best catches came from agents running the claim: Jackson probes, `jshell`, mutants. Ask every reviewer for that, not for opinions.

## How I measured

- **Time and tokens:** both Claude Code session logs, including the pipeline's 75 subagent transcripts, split at each `gh pr merge`. Active time ignores gaps over 3 minutes, which removes waits for me. It's an estimate, not exact agent time.
- **Findings:** the pipeline's retrospectives and session reports.
- **Tests:** the JUnit XML from both builds.
- **Behaviour:** both jars run locally and probed with the same requests.
- **Commits measured:** the original at v1 (`aea85a0`), the pipeline at MVP (`8a8e3b0`).
