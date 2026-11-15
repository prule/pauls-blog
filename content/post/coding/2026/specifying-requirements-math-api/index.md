---
title: "One Feature, Six Specs: How Requirements Were Written Across the math-api Builds"
date: 2026-11-15T15:00:00+11:00
publishDate: 2026-11-15
draft: false
layout: "post"
tags: ["claude-code", "openspec", "requirements", "use-cases", "ai-assisted-coding", "process", "migration"]
categories: ["coding"]
description: "The same DIVIDE feature specified six ways across seven builds of math-api, from task prompts and use cases to outcome items, OpenSpec scenarios and golden responses, and what each form actually changed."
author: "Paul"
---

The format of each backlog item mattered less than I expected. Two things mattered more:

- **Shared project docs that state rules exactly.** These were the same in every Java build, and they're why every build ended with the same design.
- **An exact rule close to the change.** Whatever form the item took, a rule only got checked when it was restated where the reviewer would read it for that change, or when it was executable.

The item format still changed three things: **who did the design, what it cost, and who decided the ambiguous cases.** Task prompts handed the design to the agent. Use cases and outcome items made the agent design, and cost time. And when the agent designed, it also chose how to read a vague word, sometimes by changing the contract.

Across seven repos, I wrote requirements in six forms. This post follows one feature, DIVIDE, through all of them, and then through the five-agent pipeline, where OpenSpec artifacts are how one agent hands work to the next.

| Form | Builds | Who designs | Most exact part |
|---|---|---|---|
| Shared docs (PRD, DOMAIN-MODEL, STANDARDS) | All Java builds | The doc author, once | BR tables with examples |
| Task prompt | math-api, one-agent-2, vibe-coded | The prompt | Classes, status codes, tests named |
| Business use case | sdlc-pipeline | The architect agent | Alternative flows |
| Outcome item with "done when" | Both outcome builds | The builder | Acceptance checks |
| OpenSpec spec delta + design.md | All OpenSpec builds | Propose step | WHEN/THEN scenarios |
| Code as the spec, plus golden responses | kotlin-migration | The old code | 375 recorded HTTP responses |

The earlier posts have the cost and quality numbers: [the summary]( {{< ref "post/coding/2026/math-api-experiments-summary/" >}} ) links to all of them.

## 0. The layer underneath: shared docs

Every Java build started from the same three documents. They're the reason the design converged. DOMAIN-MODEL has a glossary that maps each term to code, a table of edge cases per operation, and numbered business rules:

```text
| BR-2 | Division by zero is rejected: a DIVIDE or MODULO whose b is zero, and a POWER of zero
         to a negative exponent. Applies to any zero representation (0, 0.00, -0). |
| BR-3 | DIVIDE, POWER and SQRT use MathContext.DECIMAL128: 34 significant digits,
         RoundingMode.HALF_EVEN, correctly rounded ... |
```

```text
| DIVIDE | a ÷ b under BR-3 | BR-2; 1 ÷ 3 rounding; exact 1 ÷ 4 = 0.25; negative divisor |
```

These docs already name `MathContext.DECIMAL128` and `HALF_EVEN`. Whatever form the backlog item takes, the hardest decisions have been made one layer down. Every form below is a change against these docs, not a spec on its own.

## 1. Task prompt: the design, handed over

The original math-api build wrote its backlog after it had worked out the design, and one-agent-2 and vibe-coded reused that backlog. Item 3:

```text
Add DIVIDE to the Calculation capability per docs/DOMAIN-MODEL.md BR-2 and BR-3:
MathContext.DECIMAL128 (34 significant digits, HALF_EVEN), result normalised per BR-4.
Division by any zero representation (0, 0.00, -0) raises a domain DivisionByZeroException
mapped to 422 Problem Details with type /problems/division-by-zero. Test-first: 1 ÷ 3
rounding, exact 1 ÷ 4 = 0.25, negative divisor, each zero representation; web-slice test
for the 422. Should only need a new Operation constant, an exception and its mapping.
```

It names the exception, the status, the problem type and the tests, and even predicts the size of the diff. That makes it fast: the vibe build did items 2–5 in **32.6 minutes**, the quickest of any build. It also made OpenSpec's propose step mostly redundant, because propose ended up restating the prompt as four files.

**Good for:** cheap, predictable changes once you know the design. **Bad for:** finding out whether the design is right. The prompt *is* the plan, and nothing reviews it.

## 2. Business use case: behaviour only

The pipeline's use-case writer works strictly at the business level: no endpoints, classes or libraries. UC-003:

```text
## Main flow
1. The Client service asks the system to perform a Calculation with the Operation DIVIDE
   and two Operands, a (the dividend) and b (the divisor), as in UC-002.
2. The system checks the request as in UC-002 (well formed, supported Operation).
3. The system checks that b is not zero (BR-2).
4. The system divides a by b. If the quotient cannot be written exactly in 34 significant
   digits, it rounds to 34 significant digits, half to even (BR-3). ...

## Alternative & exception flows
- 3a. b is zero, in any form (0, 0.00, -0, given as a number or as text): the system
  rejects the Calculation as division by zero ...
- 3b. a is zero and b is not: this is not a rejection. The Result is 0.
- 3c. The request is both invalid and divides by zero: only the first failure is reported,
  in the order of R-1. Here that is invalid request.
```

The alternative flows are the best part. `0 ÷ 5` and "invalid *and* divides by zero" appear in no other form. Use cases make you think about the failure paths and the order they're checked in.

The cost is that someone has to turn behaviour into a design. In the pipeline, the architect took **39% of all agent time**, and most of that work was deriving what the task prompt would have said. For a feature this small, it's a lot of ceremony: DIVIDE went through five agents and two gates, and the reviews found three wording nits.

**Good for:** non-technical stakeholders, and features where the failure paths are the hard part. **Bad for:** small changes. The format makes everything as big as a user goal.

## 3. Outcome item: done-when checks

The outcome builds got this:

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

It sits between the last two forms. It names results, not classes, and each check becomes a test. "Not in this item" fences off scope creep. "Changed nothing in how the other operations behave" is a regression requirement that no other form states.

The vibe build slowed from 32.6 to **40.1 minutes**, because it now had to design. The OpenSpec build barely changed, since its propose step designed in both rounds.

The real effect was on **ambiguity**. The PRD says a body over "16,384 characters" is rejected, but Jackson counts bytes. With task prompts, that never came up. With outcome items, each builder had to settle it:

- **vibe** changed the PRD to "16,384 bytes": it changed the contract to fit the tool.
- **openspec** kept "characters" and wrote 64 lines that count code points: it changed the code to fit the contract.

The item format allowed both, and it was up to the builder. An outcome backlog hands the agent judgement calls about the contract. Some of those calls belong to a person.

**Good for:** making the builder design, with clear acceptance checks. **Bad for:** contracts with vague words in them. Settle units and boundaries first, or the agent will settle them for you.

## 4. OpenSpec: scenarios and design.md

In the OpenSpec builds, the propose step turns whichever input it gets into a proposal, a design, a spec delta and tasks. The spec is WHEN/THEN:

```text
### Requirement: Exact addition, subtraction and multiplication
POST /api/v1/calculations with operation ADD, SUBTRACT or MULTIPLY SHALL return 200 with
the exact Result of a + b, a − b or a × b, with no rounding (BR-3) ...

#### Scenario: Decimal addition is exact
- WHEN a client sends {"operation": "ADD", "a": 0.1, "b": 0.2}
- THEN the response is 200 with "result": "0.3"
```

Two parts of this did different jobs:

- **The spec scenarios** mostly restated the PRD and DOMAIN-MODEL. They became tests, which helped branch coverage in one round. They're also a third copy of the same facts. No later change broke old behaviour, so the living specs never got to prove their main value.
- **design.md** stated each rule as it would be coded. That's where the BR-1 bug was caught.

### The BR-1 lesson

BR-1 caps an Operand's scale. In **every** Java build, DOMAIN-MODEL already stated it exactly:

```text
| BR-1 | An Operand, as written, has at most 100 significant digits (precision() ≤ 100)
         and a scale within ±1000 (-1000 ≤ scale() ≤ 1000). ...
```

Five builds still wrote `Math.abs(operand.scale()) > 1000`, which lets `Integer.MIN_VALUE` through. The rule was in the project docs every time. It got checked only when it was also written somewhere closer to the change:

- **one-agent-2:** design.md said "two comparisons". Both code reviews compared the diff against it.
- **outcome openspec:** design.md said `-1000 <= scale() <= 1000`. The agent noticed its code didn't match.
- **pipeline:** a QA mutant showed that the test couldn't fail.
- **both vibe builds:** they had only DOMAIN-MODEL. Every reviewer read the line and moved on.

{{< notice type="info" >}}
A rule in the project docs isn't enough. Reviewers check a diff against what's in front of them for that change: the item, the design, the tests. To get a rule checked, restate it next to the change, or make it executable.
{{< /notice >}}

That corrects my [summary post]( {{< ref "post/coding/2026/math-api-experiments-summary/" >}} ), which said the vibe builds had "no such statement". They had one, but one layer too far away.

**Good for:** design.md, as an exact target for reviewers. **Bad for:** spec deltas that copy the domain docs. They cost tokens and add a place for drift.

## 5. Multi-agent: OpenSpec as the hand-off between agents

In the single-agent builds, OpenSpec is a plan that one session writes and then reads back. In the pipeline, it's the **only shared memory**. Each agent starts cold, and the change folder is how one agent tells the next what to do.

{{< mermaid >}}
flowchart LR
  UW["use-case-writer"] -- "UC-003.md" --> AR["architect"]
  AR -- "proposal, design,<br/>spec delta, tasks" --> SR["spec-reviewer"]
  SR -- "APPROVE /<br/>REQUEST CHANGES" --> G1{{"Gate 1<br/>(me)"}}
  G1 --> JD["junior-dev"]
  JD -- "code + ticked tasks.md" --> QA["qa"]
  QA -- "mutants vs spec delta" --> SD["senior-dev"]
  SD -- "diff vs spec delta" --> G2{{"Gate 2<br/>(me)"}}
  G2 --> A["archive +<br/>retrospective + ledger"]
{{< /mermaid >}}

Each agent works on a different part of the change folder:

| Agent | Reads | Writes | Checks against |
|---|---|---|---|
| use-case-writer | PRD, DOMAIN-MODEL | the use case | the domain docs |
| architect | use case, docs, `openspec/config.yaml` | proposal, design.md, spec delta, tasks.md | the profile and STANDARDS |
| spec-reviewer | the four artifacts | a verdict with blocking and advisory points | standards, feasibility, task quality |
| junior-dev | tasks.md, design.md | code, tests, ticked tasks, red test output | the tasks, in order |
| qa | spec delta, tasks.md, code | extra tests, mutation results | every requirement and every task |
| senior-dev | the diff, spec delta | fixes | the spec delta and STANDARDS |

`openspec/config.yaml` turned the shared docs into rules that every artifact had to follow:

```yaml
context: |
  Before writing any artifact, read docs/PRD.md, docs/DOMAIN-MODEL.md and docs/STANDARDS.md.
  They are the source of truth for scope, domain terms and business rules (BR-n) ...
rules:
  proposal:
    - State what is out of scope.
    - List which of PRD.md, DOMAIN-MODEL.md, STANDARDS.md this change must update (or "none").
  specs:
    - Use the DOMAIN-MODEL.md glossary terms exactly and cite business rules by BR-n.
  tasks:
    - Order tasks test-first (write the failing test, then the code).
```

### The plan had to carry everything

A cold implementer can't ask the architect what it meant, so the artifacts got much bigger. The task prompt for DIVIDE was **6 lines**. The pipeline's DIVIDE change was **358 lines**: a 42-line proposal, a 91-line design, a 140-line spec delta and 85 lines of tasks. Across five changes, the pipeline wrote **1,708 lines** of plans, against one-agent-2's 847.

tasks.md came out as the most exact artifact. It names every test method, input and expected value:

```text
- BR-3: dividesOneByThreeToThirtyFourSignificantDigits: "1", "3" → "0.3333333333333333333333333333333333".
- BR-3: roundsATieDownToTheEvenDigit: "12345678901234567890123456789012345", "10"
  → "1234567890123456789012345678901234".
- BR-2: rejectsZeroDividedByZero: "0", "0" throws DivisionByZeroException, not
  ArithmeticException. Assert isExactlyInstanceOf(DivisionByZeroException.class).
```

So the pipeline ends up back where the task prompt started. It begins with a use case that names no classes, and the architect turns that into a task list more prescriptive than anything I wrote by hand. The design work doesn't go away. It moves to an agent, and that agent took **39% of the pipeline's time**.

### Two rules that kept the plan true

tasks.md also carried process rules, copied in from the pipeline's ledger of lessons:

```text
Departures (ledger L-1): in these cases, update both design.md and tasks.md in the same
apply step, before QA, and say so in your report:
- you add a dependency, set a config property or create a class that design.md does not name;
- a default in design.md "Framework and JDK defaults relied on" turns out different; ...

Red evidence (ledger L-6): each failing-test task asks for the red output of
./gradlew test --tests …. Paste that output into your report before you start the
implementing task.
```

**Departures** are the important rule. In the single-agent OpenSpec builds, the agent "simplified" two comparisons into `Math.abs` without saying so, and only review caught the mismatch. Here, an implementer that departs from the plan has to change the plan first. Downstream agents check against the spec delta and design, so those files have to stay true.

QA then made the spec executable. It checked every requirement in the spec delta and broke the code on purpose: 8 mutants for DIVIDE, including half-up and half-down rounding, each caught only by its own tie test.

### What it cost and what it bought

For DIVIDE, the gates found little: 3 advisory points from the spec reviewer (a wrong cross-reference, a stale Purpose line, an unstated MockMvc detail), no QA findings, and no code-review fixes. The retrospective called it "the cleanest run so far". For operand limits, with parser internals and boundary values, the same gates raised 15 findings, including 3 failed reviews.

The single-agent builds had the same weakness with their retrospectives. The ledger lessons held only because the orchestrator pasted them into each change's tasks.md. None was ever written into its real target: `config.yaml`, STANDARDS or the agent definitions. The requirements format carried the lessons forward, but only while someone kept copying them in.

**Good for:** handing work between agents that share no context, with each one checking a different part of the plan. **Bad for:** small changes. Every hand-off needs a plan detailed enough to stand alone, and that's what makes the pipeline cost about twice as much.

## 6. Code as the spec: recovered rules and golden responses

The Kotlin migration flips the setup. The docs are **off-limits**. The Java code, tests and config are the only spec. The first artifact is a recovered behaviour catalogue, with each row read from code:

```text
- RB-6 Operand limits: precision ≤ 100, −1000 ≤ scale ≤ 1000, checked on the operand as
  written. The check uses two comparisons, so Integer.MIN_VALUE scale is rejected. ...
- RB-7 Division by any zero (0, "0.00", "-0", "0E+3") → 422 division-by-zero.
- RB-9 Error precedence: invalid-request → unknown-operation → operand-out-of-range →
  division-by-zero.
```

Those rows became OpenSpec requirements, each one citing its **evidence** in the Java repo:

```text
### Requirement: Division by zero
Dividing by zero in any representation (0, "0.00", "-0", "0E+3") SHALL be 422
division-by-zero with detail "Division by zero is undefined."

Evidence: Java domain/Operation.java:29-30, DivisionByZeroException.java:7,
ProblemDetailsAdvice.java:63-65; tests OperationTest.rejectsDivisionByEveryZeroPerBR2, ...
```

Under those sits the most exact form in the whole series: **375 golden HTTP responses**, recorded from the running Java service by a black-box contract suite that passed against Java before any Kotlin was written.

```json
"DIVIDE a=\"0.00\" b=\"-0\"" : {
  "status" : 422,
  "Content-Type" : "application/problem+json",
  "body" : {
    "detail" : "Division by zero is undefined.",
    "instance" : "/api/v1/calculations",
    "status" : 422,
    "title" : "Division by zero",
    "type" : "/problems/division-by-zero"
  }
}
```

That spec caught things no written form had:

- **Framework defaults are requirements.** 404/405/406/415 bodies, actuator exposure, keep-alive timeouts, the missing `Server` header: none was in any PRD, and all are on the wire.
- **Accidents look like features.** The Java service accepts and returns YAML, because springdoc puts the YAML module on the classpath. Nobody asked for that. Deciding to drop it needed a person, and it's logged as an accepted difference.
- **Bugs are part of the spec.** The bytes-against-characters problem from section 3 turns up again as *source defect 1*: the code counts bytes and the message says characters. The migration rule is to **port it faithfully**, pin it in a test, and fix it later as its own backlog item. Code as the spec can't tell intent from bug. All it can do is make the bug visible and give it a name.

**Good for:** proving two systems equal. It's the only form here that is executable from end to end. **Bad for:** knowing what was *meant*. Every `BR-n` comment in the Java code was kept as a hint that a rule exists, but not trusted as the spec.

## What I'd use

{{< mermaid >}}
flowchart TB
  D["Shared docs<br/>glossary, BR tables with examples, edge-case tables"] --> O["Outcome item<br/>done-when checks + 'not in this item'"]
  O --> X["Exact rules for this change<br/>(design.md or a PR 'Design' section)"]
  X --> T["Executable checks<br/>tests, contract suite, golden responses"]
  U["Use-case alternative flows"] -. "for failure-heavy features" .-> O
{{< /mermaid >}}

- **Put the most effort into the shared docs.** A glossary mapped to code, BR tables with worked examples, and per-operation edge cases. Every build's design came from these.
- **Write backlog items as outcomes**, with "done when" checks, a "not in this item" line and a regression check. Settle vague words such as "characters" and "every error" in the docs first, before the agent settles them in code.
- **Restate the exact rule next to the change.** One line in design.md or the PR, written as it'll be coded, gives the reviewer something to compare the diff with.
- **Borrow use-case alternative flows** for features where the failure paths are the hard part.
- **Make the spec executable wherever it's cheap.** Golden responses and contract tests are the only form that can't drift from what the system does.
- **Skip spec deltas that only copy the domain docs.**

## Caveats

- **One feature, one model.** Every build used Claude Opus 5.5 on the same small API. DIVIDE is simple, and a feature with more ambiguity might reward use cases more.
- **The forms weren't independent.** Each build inherited docs refined by the builds before it. The shared layer got better over time, which flatters the later forms.
- **The Kotlin migration is unfinished.** Its numbers are from items 1–6. Requirements recovered from code haven't yet been compared with the original docs (backlog item 8).
