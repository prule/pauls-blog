---
title: "With and Without OpenSpec: Building the Same API a Fourth Time"
date: 2026-11-13T19:30:00+11:00
publishDate: 2026-11-13
draft: false
layout: "post"
tags: ["claude-code", "openspec", "ponytail", "code-review", "ai-assisted-coding", "process", "sonarqube"]
categories: ["coding"]
description: "The same API built twice from the same docs and backlog, once planning each change in OpenSpec and once going straight from prompt to code, with the same reviews: cost, tests, bugs and behaviour compared through item 7."
author: "Paul"
---

With the same code reviews on both, the build that planned each change in OpenSpec took **49.6 active minutes** for items 2–5. The build that went straight from prompt to code took **32.6**. The OpenSpec build ended with **129 tests to 73** and **94% branch coverage to 80%**, and it caught a BR-1 bug that the other build shipped. On everything else the two came out level: same architecture, same SonarQube result, same status and type on 24 of 28 live probes.

That result has a big qualifier. The backlog prompts were prescriptive: they already spelled out the design that OpenSpec's propose step would otherwise have to produce. The build without OpenSpec still got a plan up front. It just came from the backlog instead of from `/opsx:propose`.

A second round through items 5a, 6 and 7 kept the same pattern. OpenSpec cost **1.45× the time** (26.5 against 18.3 minutes) and produced more tests (261 against 152). This time it caught nothing the other build shipped: both builds behave identically on all 36 new edge-case probes.

In [Reviews Without the Pipeline]( {{< ref "post/coding/2026/reviews-without-the-pipeline/" >}} ) one agent with reviews from item 1 ([math-api-one-agent-2](https://github.com/prule/math-api-one-agent-2)) beat a five-agent pipeline on cost and matched it on findings. That build still planned every change in OpenSpec first. So this time I dropped OpenSpec. Same docs and same backlog prompts, but each item goes straight from prompt to code, followed by the same ponytail review and `/code-review`, a retro, and a PR. The result is [math-api-vibe-coded](https://github.com/prule/math-api-vibe-coded). I compared the two at item 5 (OpenAPI docs), then ran both through items 5a, 6 and 7 for a second round.

| | vibe | openspec |
|---|---|---|
| Active time, items 2–5 | 32.6 min | 49.6 min |
| Output tokens, items 2–5 | 137k | 190k |
| Tool calls, items 1–5 | 189 | 271 |
| Review findings recorded | 40 | 45 |
| Tests (all green) | 73 | 129 |
| Branch coverage | 80.0% | 94.4% |
| Sonar ncloc | 478 | 338 |
| Planning docs written | 0 lines | 847 lines |
| Contract bugs left (live probes) | 1 | 1 |
| Human prompts, items 1–5 | 11 | 19 |

Both builds ran on Claude Opus 5.5. The full report, with charts and every probe, is in the repo: [reports/comparison/vibe-vs-openspec.html](https://htmlpreview.github.io/?https://github.com/prule/math-api-vibe-coded/blob/main/reports/comparison/vibe-vs-openspec.html).

## 1. The only difference is the propose step

{{< mermaid >}}
flowchart LR
  subgraph openspec
    P["/opsx:propose"] --> PR["ponytail review<br/>(plan)"] --> A["/opsx:apply"]
  end
  subgraph vibe
    B["backlog prompt"] --> C["tests + code"]
  end
  A --> R["ponytail + /code-review"]
  C --> R
  R --> F[fix findings] --> RT[retro] --> M[PR, merge on green]
{{< /mermaid >}}

Everything after the code was the same: ponytail inline, `/code-review` with its two sub-agents (standards and spec), every finding fixed or noted, a retro with the same four sections, and a squash merge on green CI. Both builds started from identical PRD, DOMAIN-MODEL and STANDARDS, and got the same implementation-level backlog prompts. The vibe copy only lost the `/opsx:propose` prefix.

The time and token gap is all planning. Proposing, the plan review, validation and archiving added **3–6 active minutes and 8–18k output tokens to every item**. The review sub-agents cost the same in both builds: 10 runs each.

## 2. The backlog was already a plan

The backlog came from the original math-api build, where it was written after the design had been worked out. Each item's prompt reads like the output of a propose step. Here's item 3:

```text
Add DIVIDE to the Calculation capability per docs/DOMAIN-MODEL.md BR-2 and BR-3:
MathContext.DECIMAL128 (34 significant digits, HALF_EVEN), result normalised per BR-4.
Division by any zero representation (0, 0.00, -0) raises a domain DivisionByZeroException
mapped to 422 Problem Details with type /problems/division-by-zero. Test-first: 1 ÷ 3
rounding, exact 1 ÷ 4 = 0.25, negative divisor, each zero representation; web-slice test
for the 422. Should only need a new Operation constant, an exception and its mapping.
```

That prompt names the classes, the exception, the status code, the rounding mode and the test cases. It is the design. In the OpenSpec build, `/opsx:propose` mostly restated it as proposal, design, spec delta and tasks. In the vibe build, I fed it straight to the agent.

So this experiment doesn't compare "with a plan" against "without a plan". It compares a plan that was written down once more against a plan used as given. In a real project the backlog item would be closer to "add division" and the propose step would have to do the design work itself. That is where OpenSpec should earn more than it could here.

## 3. The bug that only the design caught

BR-1 caps an Operand at 100 significant digits and a scale within ±1000. Both builds wrote the scale check like this:

```java
if (operand.precision() > MAX_PRECISION || Math.abs(operand.scale()) > MAX_SCALE) {
    throw new OperandOutOfRangeException(name);
}
```

`new BigDecimal("1E+2147483648")` has scale `Integer.MIN_VALUE`. `Math.abs(Integer.MIN_VALUE)` is still `Integer.MIN_VALUE`, which is negative, so the operand passes. In the vibe build the request then dies inside `BigDecimal` with `ArithmeticException: Overflow`, and the client gets a **500** where the PRD promises a 400 `operand-out-of-range`. The catch-all handler keeps internals out of the response and the server does no heavy work, but it's a contract bug.

The OpenSpec build's design.md said "two comparisons". The agent "simplified" them to `Math.abs` while implementing, and **both code reviews flagged the mismatch between design and code**. The vibe build had no design to diverge from. Ponytail, the standards reviewer and the spec reviewer all read the `Math.abs` line and moved on.

{{< notice type="info" >}}
The model made the same mistake in both builds. What differed was whether a reviewer had an exact statement of intent to compare the code against. That's the clearest thing OpenSpec bought here.
{{< /notice >}}

## 4. Without a written design, the spec reviewer did the work

The vibe build's code reviews recorded 40 findings, against the OpenSpec build's 45. Ten of those 45 came from the OpenSpec plan review. Almost all of the vibe build's findings came from `/code-review`, and the spec reviewer did most of it. It ran the real jar and compared responses with the PRD, and it found every contract bug the vibe build fixed:

- **Item 2.** Jackson converts `{"operation": 123}` into the string `"123"`, which returned unknown-operation instead of invalid-request.
- **Item 4.** A 1,001-character field name was accepted, because Jackson's `maxNameLength` is separate from `maxStringLength`. The body limit counts bytes, not characters, so a body full of `é` was rejected early.
- **Item 5.** The generated `ProblemDetail` schema had a phantom `properties` field. The examples were missing `instance`, and 406, 415 and 500 weren't documented.

Ponytail found nothing in items 1–3. Its only finding of its own was a minor error-type edge, and it shared one more finding with the other reviewers. It asks "is this minimal and correct?", and with no plan to judge, that question had less to bite on.

Two of those fixes leave the vibe build **stricter than the OpenSpec build** on live probes:

| Request | vibe | openspec |
|---|---|---|
| `"a":"1E+2147483648"` | **500** (BR-1 bypassed) | 400 invalid-request |
| Field name of 1,001 characters | 400 invalid-request | **200** |
| `"operation": 123` | 400 invalid-request | 400 unknown-operation, echoing `'123'` |
| Missing field(s) | Detail names the required fields | One generic detail, which says "16,384 characters" though the parser counts bytes |

The other 24 probes returned the same status and type.

## 5. SonarQube calls it nearly a draw

I scanned both trees with SonarQube 26.9, using the default "Sonar way" profile with JaCoCo coverage:

| | vibe | openspec |
|---|---|---|
| Quality gate | Passed | Passed |
| Vulnerabilities / hotspots | 0 / 0 | 0 / 0 |
| Bugs | 1 | 1 |
| Code smells / debt | 3 / 24 min | 0 / 0 |
| Duplication | 0.0% | 0.0% |
| Line / branch coverage | 97.4% / 80.0% | 98.2% / 94.4% |

The single "bug" in each is the same false positive (S2259 on `getType() == null ? … : getType()`). The vibe build's three smells are the literals "title", "status" and "detail" repeated in its OpenAPI config.

The branch coverage gap is the real difference, and it comes from OpenSpec's scenarios: each spec scenario became a test. Coverage also showed something the reviews missed. The vibe build's `OperandDeserializer` has a 1,000-character check that **never runs during the tests**, even though the test for it passes. Jackson rejects the input first, so the check is either dead code or a backstop that nothing tests.

The vibe build also has 41% more production code. Most of it is OpenAPI and request-parsing machinery: a `ProblemType` catalogue, a custom deserializer and a Jackson config class. The reviews pushed for that structure to stop docs drifting from code. It isn't speculative, but it is bigger.

## 6. Documentation is part of the answer

OpenSpec also leaves documentation behind, and it's worth two different amounts:

- **Living specs: low value here.** `openspec/specs/` has 276 lines and 51 WHEN/THEN scenarios. They mostly restate PRD §3.2 and DOMAIN-MODEL §3, which both builds already keep current, and the tests encode the same cases. That makes a third copy of the same facts, and another place for drift.
- **Design rationale: high value.** 240 lines of `design.md` across five changes, recording why each decision was made and what was rejected, inside the repo. The vibe build has that only in PR descriptions and retros. As section 3 showed, the design isn't just for future readers. It gives reviewers an exact statement to check against.

Living specs should pay off when a change touches existing behaviour. Items 1–5 mostly added new behaviour, so round 2 was the test.

## 7. Round 2: items 5a, 6 and 7

Item 5a turns the docs off under a `prod` profile. Item 6 adds POWER and MODULO, and modifies an existing spec scenario. Item 7 adds unary SQRT and NEGATE, and its prompt leaves the arity design open. Each build ran the three items in a new session with no context, following its own CLAUDE.md through to merge on green.

| Items 5a–7 | vibe | openspec |
|---|---|---|
| Active time | 18.3 min | 26.5 min |
| Output tokens | 84k | 130k |
| Tests at item 7 | 152 | 261 |
| Branch coverage | 91.7% | 97.9% |
| Review findings | ≈22 | 21 (4 from plan review) |
| New-behaviour probes correct | 36/36 | 36/36 |
| Regressions in item 1–5 probes | 0 | 0 |

**The plan review made tests stronger but prevented no bugs.** It raised four points: a webjar 404 check that could pass for the wrong reason, BR-6 "after normalisation" untested (`1.00^999` must succeed), a worst-case scenario with no expected result, and a missing task for unary responses that would have thrown a NullPointerException. The vibe build got all four right without the review. Every POWER, MODULO, SQRT, NEGATE and arity case in DOMAIN-MODEL §3 returns the same answer from both jars.

**Ponytail made its first real code-level catch.** In the vibe build, the "exact power, round once" test couldn't fail. `BigDecimal.pow(n, DECIMAL128)` passes it, and is sometimes wrong in the last digit. Ponytail searched for an input that tells the two apart, and found `0.2857142857142857142857142857142857^668`.

**The open design decision wasn't open.** Item 7 asks where arity lives and whether `b` becomes optional. Both builds chose the same answer: `arity()` on the enum, `b` null for unary Operations, BR-9 checked before BR-1, and `b` left out of unary responses. That's because DOMAIN-MODEL §2 already describes the finished design. It's section 2's problem again, coming from the docs instead of the backlog.

**No regressions to prevent.** I reran the original 28 probes against both new jars. Status and type are unchanged in both, apart from my old "unknown operation" probe, which used POWER, now a real operation. The tests guarded old behaviour in both builds, so the living specs never got to show their main benefit.

**The milestone audit is OpenSpec's extra step.** The OpenSpec build's CLAUDE.md runs a whole-repo audit before the first post-v1 item. It flagged the same input class as the BR-1 bug (`1e2147483648`), and skipped it as a nit, because in that build the input already returns 400. The vibe build has no audit, and its 500 for that input survived three more items.

{{< notice type="note" >}}
The vibe session stopped twice with reviewers or CI still running, and needed a nudge to carry on. The OpenSpec session didn't. The OpenSpec build's CLAUDE.md says "run each step yourself, without waiting to be asked", and the vibe build's doesn't. That's a wording difference, not an OpenSpec feature.
{{< /notice >}}

## What I'd do next

- **Rerun with outcome-level backlog items**, such as "clients can divide two numbers", so the propose step has to do the design work. Building both ways from those would give a fairer read on OpenSpec.
- **Fix the vibe build's BR-1 check**, by comparing scale on both sides (`scale < -1000 || scale > 1000`) with a test at `1E+2147483648`.
- **Try a cheap middle ground.** Add a short "Design" section to each PR description that states the rules and conditions exactly as they'll be coded, and have the spec reviewer check the code against it. That may get most of what design.md gave the reviewers without a separate spec tree.
- **Add "test every numeric guard at its type limits"** to the spec reviewer's brief. It's the one class of bug the vibe build's reviews missed.
- **Add a milestone audit to the vibe process**, and "run every step without waiting" to its CLAUDE.md. Both are process wording, not OpenSpec features.

## Caveats

- **One run each.** The same model wrote the same bug in both builds, and catching it or missing it could be luck. Treat the BR-1 result as one data point.
- **The backlog did the propose step's job.** As section 2 explains, the prompts already contained the design, so both builds started each item with a plan. This is the biggest limit on what the comparison can say about OpenSpec.
- **Findings were counted differently.** The vibe tally includes judgement calls that weren't acted on, and the OpenSpec tally comes from its retros.
- **Same model, shared docs.** The vibe build came after the OpenSpec one, with the same model and docs. It didn't see the OpenSpec code or report until the comparison, but the docs already held many of that build's decisions.

## How I measured

I measured both session logs with one script, splitting at each `gh pr merge`. Active time adds up the gaps between log events from the main session and every subagent, and skips gaps over 3 minutes. Tokens are counted once per API message. The script reproduces the earlier report's figure for the OpenSpec build's items 2–5 (49.6 minutes against the published 50).

Item 1 is left out of the time comparison because the OpenSpec log includes setup there.

Round 2 used the same script on two fresh sessions, one per repo.

For behaviour, I sent the same 28 HTTP requests to both boot jars. Round 2 added 36 requests covering the new operations, plus the `prod` profile, and reran the original 28 to look for regressions. For static analysis, I ran fresh `./gradlew clean test` runs with JaCoCo added through an init script, then SonarQube 26.9 on both trees at `d9378cf` and `f257be0`.
