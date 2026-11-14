---
title: "With and Without OpenSpec, Part 2: An Outcome Backlog"
date: 2026-11-14T10:30:00+11:00
publishDate: 2026-11-14
draft: true
layout: "post"
tags: ["claude-code", "openspec", "ponytail", "code-review", "ai-assisted-coding", "process", "sonarqube"]
categories: ["coding"]
description: "The same API built twice again, this time from a backlog of outcomes rather than designs: OpenSpec's time overhead fell from 52% to 16%, and once more it was the only build to catch the same BR-1 bug."
author: "Paul"
---

When the backlog states outcomes and leaves the design to the builder, OpenSpec's extra cost in time drops from **52% to 16%**: 46.4 active minutes against 40.1 for items 2–5. Its extra cost in tokens doesn't drop. It still writes **1.43× the output tokens**. Most of the time gap closed because the vibe build slowed down: it took 32.6 minutes with the task backlog and 40.1 now that it has to design for itself.

The quality result points the same way as last time, but the margin is thin. Both builds wrote the same BR-1 bug again, and only the OpenSpec build fixed it. OpenSpec also follows the PRD more literally. The vibe build came out smaller, with **95% branch coverage against 82.5%**, and with better-documented error responses. SonarQube passed both, with one code smell each.

[Last time]( {{< ref "post/coding/2026/vibe-coding-vs-openspec/" >}} ) the main caveat was that the backlog was already a plan. Every prompt named the classes, status codes and test cases, so OpenSpec's propose step had little left to do. The first item on that post's "what I'd do next" list was to rerun with outcome-level backlog items. This is that rerun.

| | vibe | openspec |
|---|---|---|
| Active time, items 2–5 | 40.1 min | 46.4 min |
| Output tokens, items 2–5 | 137k | 197k |
| Tool calls, items 1–5 | 213 | 302 |
| Review findings recorded | 68 | 74 |
| Tests (all green) | 102 | 127 |
| Branch coverage | 95.0% | 82.5% |
| Sonar ncloc | 474 | 555 |
| Planning docs written | 0 lines | 983 lines |
| Live probes matching (status + type) | 38 / 45 | 38 / 45 |
| Human prompts, items 1–5 | 14 | 11 |

Both builds ran on Claude Opus 5.5:

- [math-api-outcome-based-backlog-vibe](https://github.com/prule/math-api-outcome-based-backlog-vibe)
- [math-api-outcome-based-backlog-openspec](https://github.com/prule/math-api-outcome-based-backlog-openspec)

The full report, with charts and every probe, is in the OpenSpec repo: [reports/comparison/outcome-backlog-vibe-vs-openspec.html](https://htmlpreview.github.io/?https://github.com/prule/math-api-outcome-based-backlog-openspec/blob/main/reports/comparison/outcome-backlog-vibe-vs-openspec.html).

## 1. Same docs, outcome-shaped backlog

The PRD, DOMAIN-MODEL and STANDARDS are the same as before. The backlog changed. Here's item 3 now:

```text
### 3. Clients can divide

- Outcome: A client can divide two operands and gets a result rounded the one documented
  way, or a clear rejection when dividing by zero.
- We know it's done when:
  - 1 ÷ 3 returns 34 significant digits, rounded per BR-3; 1 ÷ 4 returns "0.25".
  - Dividing by any representation of zero (0, 0.00, -0) returns 422 per BR-2.
  - Adding DIVIDE changed nothing in how the other operations behave.
- Not in this item: client-chosen precision.
```

The item names no classes, exceptions or test layout. Each "done when" check needs a test, and the builder chooses the design within STANDARDS. My prompt for every item was just `start item N`. Each repo's CLAUDE.md drives the rest, through to the merge.

{{< mermaid >}}
flowchart LR
  subgraph openspec
    P["/opsx:propose"] --> A["/opsx:apply"] --> V["/opsx:verify"]
  end
  subgraph vibe
    B["outcome"] --> C["design + tests + code"]
  end
  V --> R["ponytail + /code-review"]
  C --> R
  R --> F[fix findings] --> M[retro, PR, merge on green]
  M -. openspec only .-> AR["/opsx:archive"]
{{< /mermaid >}}

This OpenSpec process has **no plan review**. The earlier OpenSpec build ran ponytail over each proposal before apply. This one goes straight from propose to apply and adds `/opsx:verify` after. Section 5 shows what that cost.

## 2. The vibe build paid for design this time

| Items 2–5 | vibe | openspec | time | tokens |
|---|---|---|---|---|
| Task backlog (design in the prompt) | 32.6 min | 49.6 min | 1.52× | 1.39× |
| Outcome backlog (builder designs) | 40.1 min | 46.4 min | 1.16× | 1.43× |

The vibe build got **23% slower**. The OpenSpec build barely changed. That makes sense: its propose step did design work in both rounds. With the task backlog, that work mostly restated a design the prompt already gave.

Output tokens didn't converge. OpenSpec still writes about 40% more text. It's mostly proposal, design, spec delta and task files, plus `/opsx:verify` and archive running in the main session. The review sub-agents cost the same in both builds: two per item, ten each.

## 3. `Math.abs(scale)`, again

BR-1 caps an Operand's scale at ±1000. This is the vibe build's check, shipped in `Calculation`:

```java
if (Math.abs(operand.scale()) > MAX_SCALE) {
    throw new OperandOutOfRangeException(name, "has a scale outside ±" + MAX_SCALE);
}
```

`new BigDecimal("1E+2147483648")` has scale `Integer.MIN_VALUE`, and `Math.abs` of that is still negative, so the check passes. **Four builds of this API now, and all four wrote that line.**

The OpenSpec build's `design.md` stated the rule as `-1000 <= scale() <= 1000`. The agent wrote `Math.abs` anyway. Seconds after its domain tests passed, its own note said "I found a bug in my own code". It then confirmed the bug with a one-line Java program and fixed it with a test:

```java
if (operand.precision() > MAX_DIGITS || operand.scale() < -MAX_SCALE || operand.scale() > MAX_SCALE) {
    throw new OperandOutOfRangeException(name);
}
```

Last round the code reviews caught it, by comparing the code against design.md. This round the agent caught it before review. None of the vibe build's three reviews flagged it, in either round.

{{< notice type="info" >}}
The vibe build's bug can't be reached over HTTP on this stack. Jackson 3 rejects `1E+2147483648` while parsing, as a string or as a number, so both jars return 400. It's a latent domain bug. In the task-backlog build the same input got through to the domain and returned a 500.
{{< /notice >}}

Two rounds of one run each don't make a rate. Both times, though, the build with the rule written down as two comparisons was the one that noticed its code said something else.

## 4. Literal vs pragmatic reading of the PRD

I sent the same 45 HTTP requests to both jars. Status and Problem Details `type` matched on 38. The 7 differences come down to how each build read the PRD:

| Request | vibe | openspec |
|---|---|---|
| 404, 405, 406, 415 | No `type` field | `"type": "about:blank"` |
| `"operation": 123` or `true` | 400 unknown-operation | 400 invalid-request |
| 14,925 chars / 24,925 bytes, non-ASCII | 400 "larger than 16384 bytes" | 200 |

**Framework errors.** PRD §3.2 says every error has a `type`. The vibe build left it out on purpose. RFC 9457 says an absent `type` means `about:blank`, which made it a reasoned call, but not what the PRD says. The OpenSpec build found the gap indirectly. `/opsx:verify` asked for the 405 and 415 bodies to go through the same Problem Details checks as every other error, and the tightened tests failed on `$.type`.

**Coercion.** Jackson converts `123` into the string `"123"` by default. The OpenSpec spec reviewer caught this against the running app. Last round, it was the vibe build's spec reviewer that caught it. Both reviewers had the same brief, so this one comes down to chance.

**The body limit is the interesting one.** The PRD says "a body over 16,384 characters". Jackson's document limit counts bytes.

- **vibe** changed the PRD to "16,384 bytes" and set one yaml block:

  ```yaml
  spring:
    jackson:
      factory:
        constraints:
          read:
            max-document-length: 16384
            max-string-length: 1000
            max-number-length: 1000
            max-nesting-depth: 500
  ```

- **openspec** kept "characters" and wrote `RequestBodyLimit`, 64 lines. It reads up to 64 KB of the body and counts code points in the declared charset. Two spec-reviewer findings got it there: one about counting UTF-16 units, and one about decoding as UTF-8 regardless of the charset.

Both moves are allowed by CLAUDE.md, which says docs may change in the same PR, and both match their own PRD. They are different kinds of decision, though. The vibe build changed the contract to fit the tool. The OpenSpec build changed the code to fit the contract. With an outcome backlog, that choice is up to the builder.

The vibe build's error details are more specific, for example "Missing or empty field(s): 'b'" and "has more than 100 significant digits". The OpenSpec build gives one BR-1 detail that lists both limits.

## 5. The plan was corrected by review, not the reverse

The OpenSpec retros have a "Spec drift" section. The plan changed in 4 of 5 items:

| Item | Added after the plan | Found by |
|---|---|---|
| 1 | Liveness/readiness requirement (Boot 4 gives it free; the design called it a non-goal) | Standards reviewer |
| 2 | 3 scenarios: operation not a string, unexpected failure, framework errors carry a type | Spec reviewer, ponytail, verify |
| 4 | 3 scenarios: code points, declared charset, huge body not read whole | Spec reviewer |
| 5 | 4 requirements, including "the rules are stated, not referenced" | Spec reviewer |

Eleven scenarios were added after code review. The OpenSpec item 4 retro admits it: *"Characters" in a contract needs pinning down … at propose time, not in review.* With no plan review, the plan is only checked once the code exists. At that point it's documentation of what review decided, not a spec the code was checked against.

**`/opsx:verify` added little.** It made 8 findings across five items:

- **5 were false alarms:** "the CI task isn't ticked", which is true of every item until a PR exists.
- **2 were minor:** a hand check standing in for a formatting test, and a redundant ArchUnit rule.
- **1 mattered:** the 405/415 assertion gap that led to the missing `type`.

The process also had an ordering bug. Archive needs the CI task ticked, and CI only runs once there's a PR. The OpenSpec build found it on item 1 and fixed it in a process PR: open a draft PR before archive.

## 6. SonarQube: both clean, and the size result flipped

SonarQube 26.9, "Sonar way" profile (599 rules), JaCoCo coverage:

| | vibe | openspec |
|---|---|---|
| Quality gate | Passed | Passed |
| Bugs / vulnerabilities / hotspots | 0 / 0 / 0 | 0 / 0 / 0 |
| Code smells / debt | 1 / 8 min | 1 / 20 min |
| Duplication | 0.0% | 0.0% |
| ncloc | 474 | 555 |
| Line / branch coverage | 98.1% / 95.0% | 98.8% / 82.5% |
| Cyclomatic / cognitive | 49 / 10 | 69 / 24 |

Last round the vibe build was 41% bigger and OpenSpec led on branch coverage, 94% to 80%. Now OpenSpec is 17% bigger and behind on branches. The extra code is `RequestBodyLimit`, `JacksonConfig` and an `arity()` that no Operation uses yet. OpenSpec added `arity()` because DOMAIN-MODEL lists it. Vibe deferred it to item 7, YAGNI.

OpenSpec's 7 missed branches are all defensive code or not yet used:

- the unary path
- the charset fallbacks
- three guard branches in the exception advice

Vibe misses a single branch, a fallback its retro says only a Jackson upgrade would hit. Neither gap hides a known bug.

The one smell in each: vibe repeats `"/api/v1/calculations"` three times in its OpenAPI config, and OpenSpec hard-codes a URI.

## 7. Docs: two readings of "from the docs alone"

Item 5's outcome is "Developers can integrate from the API docs alone".

- **openspec**: its spec reviewer read it literally. The docs had pointed readers to "BR-3" and "PRD §3.2 order", which someone without the repo can't follow. Now the endpoint description states each rule, with every number taken from the code that enforces it. A few BR ids are still left in example summaries, and the test claiming none remain only checks the description. It documents 200, 400 and 422. The 404, 405, 415 and 500 responses get only a line of prose.
- **vibe**: it documents 200, 400, 406, 415, 422 and 500. Each example is replayed against the live API and compared in full, and that test caught a wrong 415 example on its first run. Rules are stated in schema descriptions, generated from domain constants, with BR ids as tags beside them.

Both reviews caught the same hard STANDARDS §4 violation: BR-1 limits copied into annotations. Both fixed it the same way.

## What I'd do next

- **Bring back a plan review.** It's the obvious gap. Have it ask propose to settle units and boundaries, such as "characters", int ranges and what "every error" covers, before code exists.
- **Fix verify's false alarm.** Either move the CI task out of tasks.md, or run verify after the draft PR's check is green.
- **Fix the vibe build's BR-1 check** with a two-sided range and a domain test at `1E+2147483648`. Add `type` to framework errors, or write the RFC reading into the PRD.
- **Make "probe the running app with hostile input" a process step.** Both builds learned it again in item 2 and wrote it in their retros. Neither turned it into a step, so each session relearns it.
- **Test the cheap middle ground from last time.** In the vibe build, add a "Design" section to each PR that states rules exactly as they'll be coded, and have the spec reviewer check against it. The BR-1 result says the written rule is the part that matters.

## Caveats

- **One run each.** The coercion catch went to vibe last round and to OpenSpec this round, with the same reviewer brief. Single results are luck until repeated.
- **Not a clean sample.** Both outcome builds came after two task-backlog builds from the same docs, and those docs carry many of the earlier builds' decisions.
- **Session shapes differ.** The vibe session paused 8.5 hours before item 5, so its prompt cache expired. Item 1 in both builds includes one-off process and repo setup.
- **Finding counts are bullet counts** from the retros. A finding raised by two reviewers counts twice.

## How I measured

I measured both session logs with the same script as last time, splitting each log at the user's `start item N` prompts so that setup is excluded from both. Active time adds up the gaps between log events from the main session and every subagent, and skips gaps over 3 minutes. Tokens are counted once per API message.

For behaviour, I sent 45 identical HTTP requests to both boot jars and fetched the served `/v3/api-docs` from each. For static analysis, I ran fresh `./gradlew clean build` runs with JaCoCo added through an init script, because neither build includes it, then SonarQube 26.9 on both trees at `d0510d5` (vibe) and `28bd284` (openspec).
