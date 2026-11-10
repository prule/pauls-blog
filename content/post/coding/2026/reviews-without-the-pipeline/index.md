---
title: "Reviews Without the Pipeline: Building the Same API a Third Time"
date: 2026-11-10T14:00:00+11:00
publishDate: 2026-11-10
draft: false
layout: "post"
tags: ["claude-code", "openspec", "ponytail", "code-review", "multi-agent", "sdlc-pipeline", "ai-assisted-coding", "process"]
categories: ["coding"]
description: "A third build of math-api: one agent, with plan and code reviews from the first change. It cost 8% more than no reviews, found more issues than the five-agent pipeline, and ran at about 40% of the pipeline's cost."
author: "Paul"
---

Reviews are the cheap part. One agent running a plan review and two code reviews on every change cost **about 8% more** time and tokens than one agent with no reviews. It recorded **45 findings**, more than the five-agent pipeline's 36, at **about 40% of the pipeline's cost**. The pipeline still wins on one thing: it's the only build whose tests are proven to catch broken code.

In [One Agent vs Five]( {{< ref "post/coding/2026/one-agent-vs-five/" >}} ) I built [math-api](https://github.com/prule/math-api) twice. The pipeline cost 2× the time and 2.6× the tokens and bought verification, not a better design. But that comparison was lopsided: the single agent had no review step until after v1. So I ran it a third time. One agent again, but following the [CLAUDE.md process]( {{< ref "post/coding/2026/claude-md-lightweight-process/" >}} ) with every review running from item 1. It started from the same final docs as the pipeline and stopped at the same point, item 5 (OpenAPI docs and Swagger UI).

| | math-api<br/>no reviews | one-agent-2<br/>reviews | sdlc-pipeline<br/>five agents |
|---|---|---|---|
| Active time, items 2–5 | 47 min | 50 min | 112 min |
| Output tokens, items 1–5 | 217k | 235k | 562k |
| Cache writes | 367k | 957k | 2.68M |
| Tool calls | 327 | 251 | 627 |
| Review findings recorded | 0 | 45 | 36 |
| Lessons recorded | 0 | 19 | 23 |
| Tests (all green) | 108 | 129 | 155 |
| Production Java | 470 lines | 432 lines | 473 lines |

All three ran on Claude Opus 5.5 at medium effort. Item 1 is left out of the time row because math-api's item 1 also includes project setup.

## 1. Three reviews per change cost almost nothing

Each change in one-agent-2 went through:

{{< mermaid >}}
flowchart LR
  P["/opsx:propose"] --> PR["ponytail review<br/>(plan)"]
  PR --> A["/opsx:apply"]
  A --> CR1["ponytail review<br/>(diff)"]
  A --> CR2["/code-review<br/>(standards + spec)"]
  CR1 --> AR[archive + retrospective]
  CR2 --> AR
  AR --> M[PR, merge on green]
{{< /mermaid >}}

Ponytail runs inline in the main session. `/code-review` starts two short-lived subagents, one checking the diff against STANDARDS and one against the spec delta. Across five changes those ten subagents took **7 minutes and 33k output tokens** in total.

Output tokens went up 8%. Cache reads and tool calls actually went *down* compared with no reviews. A plan that's been reviewed needs less fumbling during apply. Cache writes went up 2.6×, because every reviewer subagent reads the docs and code from cold. That's the same effect that gave the pipeline 7.3×, on a smaller scale.

Where the pipeline's 92 agent-minutes went, items 1–5:

| Agent | Minutes | Output tokens |
|---|---|---|
| architect | 37.6 | 195k |
| junior-dev | 21.2 | 106k |
| qa | 16.7 | 64k |
| spec-reviewer | 10.7 | 51k |
| senior-dev | 6.2 | 29k |

The pipeline's two *review* agents took 17 minutes, about what one-agent-2 spent on all its reviews. The rest of the pipeline's cost is planning from business-level use cases, implementing in a fresh context, and QA. Reviews aren't what makes the pipeline expensive.

## 2. Two code reviews, different questions, little overlap

| Change | Plan | Ponytail code | `/code-review` | Both | Total |
|---|---|---|---|---|---|
| Walking skeleton | 4 | 1 | 7 | 0 | 12 |
| ADD / SUBTRACT / MULTIPLY | 2 | 2 | 7 | 0 | 11 |
| DIVIDE | 2 | 1 | 3 | 1 | 7 |
| Operand limits | 1 | 0 | 3 | 1 | 5 |
| OpenAPI docs | 1 | 0 | 8 | 1 | 10 |
| **Total** | **10** | **4** | **28** | **3** | **45** |

Only 3 of 45 came from both code reviews. Ponytail asks whether the code is right and minimal. `/code-review` asks whether it matches what the repo has written down, and it found the most. Many of its findings were drift: STANDARDS §2 described a looser layer rule than the ArchUnit test enforced, the proposal named the wrong class, the spec didn't cite its BR ids.

The catches that mattered:

- **A path that escaped Problem Details** (item 2, plan review). The design said no code path throws an unexpected exception. `1e-2000000000 × 1e-2000000000` throws `ArithmeticException` on scale underflow. A catch-all 500 handler went in. Neither of the other two builds has one.
- **`Math.abs(Integer.MIN_VALUE)` let an Operand past BR-1** (item 4, both code reviews). The design specified two comparisons. While implementing, the agent "simplified" them:

```java
// as written during apply: Math.abs(Integer.MIN_VALUE) is negative, so this passes
if (Math.abs(operand.scale()) > MAX_SCALE) { ... }

// as designed, and as fixed
if (operand.scale() > MAX_SCALE || operand.scale() < -MAX_SCALE) { ... }
```

The pipeline made the same mistake, and its QA mutation run caught it. math-api avoided it by casting to `long`.

- **Tests that rebuilt the expected value with the production expression** (item 3, `/code-review`). Both tests used the same `joining(", ")` as the code, so a wrong separator would still pass. They now parse the real message.
- **Docs examples that weren't the whole body** (item 5, `/code-review`). The test compared four of five fields. Now it compares the full example against a live response. The review also removed a `properties` map from the schema that never appears on the wire.

## 3. Same design a third time

The three codebases are still near triplets: the same `domain / application / adapter.in.web / config` layering under ArchUnit, an `Operation` enum, BR-1 in the `Calculation` constructor, one `ResponseEntityExceptionHandler` advice, and the same Jackson read limits. one-agent-2 has the smallest production code (432 lines) and the leanest plans (847 lines against 1,136 and 1,708).

Same edge cases sent to the three jars:

| Case | math-api | one-agent-2 | pipeline |
|---|---|---|---|
| 405 / 415 framework errors | **Missing `type`** | `about:blank`, tested for 404/405/406/415 | `about:blank`, tested |
| Unexpected exception | No handler | 500 Problem Details | No handler |
| OpenAPI examples | Hand-written | Full body checked against a live response | Checked against a live response |
| Tests proven to catch broken code | No | By hand, per finding | 8–21 mutants per change |
| Several required fields missing | Names every field | Names none | Names one |
| Operand out of range | Says which limit | States both | States both |
| Deeply nested 1.2 KB body | Generic | Generic, states both limits | Wrongly says "too large" |

The reviews closed the gaps the original had: missing `type`, hand-written examples. They didn't add the error detail math-api has on missing fields and limits. Nobody asked for it, so no review raised it.

## 4. The pipeline's real edge is QA by mutation

one-agent-2's rule was "every review point gets a test that fails without the fix". It checked that by hand for each finding. The pipeline's QA agent broke every change on purpose, 8–21 times, and closed every gap it found. That's the one thing the pipeline does that the reviewed single agent didn't.

It's also cheap to add: a mutation pass in the code-review step, run on a scratch copy. That would likely close most of the remaining gap for a fraction of the pipeline's cost.

## 5. Retrospectives repeat, they don't compound

one-agent-2 wrote five retrospectives: 19 lessons, 13 surprises, 2 `shortcut:` comments added and both paid off by item 4. The same lesson kept coming back:

> Check the framework's defaults before planning.

It came up in four of the five changes: the `/actuator` index Boot serves, Spring 7 leaving `ProblemDetail.type` null, Jackson's 1,000-character number limit, a springdoc setting that changed nothing. Each time the plan review or a failing test caught it. Each time the lesson was written down, and the next propose step made the same kind of mistake.

That's the same failure as the pipeline's ledger. Lessons only stick when they go into the file the next step reads: the propose prompt, `openspec/config.yaml`, STANDARDS. Writing them in a retrospective is only half the job.

{{< notice type="warning" >}}
**What makes this comparison unfair**

- one-agent-2 and the pipeline both started from math-api's *final* docs. math-api wrote its docs as it went.
- one-agent-2 used math-api's implementation-level backlog prompts. The pipeline's use cases left the design to its architect, which is part of why planning took 40% of its agent time.
- one-agent-2's retrospectives count every point, including naming and doc fixes. The pipeline counted what its gates reported. "45 against 36" says more about how much each looked at than about bugs found.
- one-agent-2 stopped after every plan review to ask "fix?", though CLAUDE.md says to apply review points without waiting. That's 5 of its 17 human prompts.
{{< /notice >}}

## What I'd do next

- **Default to one agent with reviews.** It got most of the pipeline's benefit for a small overhead.
- **Add mutation testing to the code-review step.** It's the pipeline's one clear advantage.
- **Write recurring lessons into the files they target,** starting with "check framework defaults" in the propose prompt.
- **Keep both code reviews.** They overlap little: ponytail finds wrong code, `/code-review` finds drift from the written standards and spec.
- **Save the pipeline for changes where a separate plan and QA pass is worth 2.4× the time.**

## How I measured

- **Time and tokens:** the three Claude Code session logs, including every subagent transcript, measured with one script and split at each `gh pr merge`. Active time adds up the gaps between log events across the main session and subagents, and skips any gap over 3 minutes, which removes waits for me. Tokens are counted once per API message. The script reproduces the earlier post's token totals exactly.
- **Findings, lessons, surprises:** each change's `retrospective.md`, and the pipeline's session reports.
- **Tests:** JUnit XML from each build.
- **Behaviour:** the one-agent-2 jar probed locally. The math-api and pipeline rows come from the probes in the earlier post.
- **Commits measured:** math-api `aea85a0`, one-agent-2 `f257be0`, pipeline `8a8e3b0`, each at item 5.
