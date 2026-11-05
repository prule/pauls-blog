---
title: "A Lightweight Process in One CLAUDE.md"
date: 2026-11-05T12:00:00+11:00
publishDate: 2026-11-05
draft: false
layout: "post"
tags: ["claude-code", "openspec", "ponytail", "sonarqube", "ai-assisted-coding", "process"]
categories: ["coding"]
description: "How a ten-step CLAUDE.md turns OpenSpec, ponytail reviews, SonarQube and a retrospective into a process the agent runs end to end on its own."
author: "Paul"
---

The process for Strength v2 (a local-first PWA for logging barbell training) is one section of `CLAUDE.md`. Ten numbered steps. Each step has a tool and a "done when". The agent runs all ten without being asked, and stops only when a step needs my decision.

No plugin orchestration, no agent swarm. Just:

- **[OpenSpec]( {{< ref "post/coding/2026/open-spec/" >}} )** for the plan and the specs.
- **ponytail-review** as the gate, twice: on the plan, then on the code.
- **The usual checks** (Vitest coverage, `tsc`, Prettier, oxlint, Playwright) plus **SonarQube** through a local `sonar-check` script.
- **A retrospective** per change, so the next change starts smarter.
- **Branch, two commits, PR, merge on green.**

The repo has archived 31 OpenSpec changes since 22 October; the full process has run since 1 November. The reviews have caught a restore that could wipe all data and an e2e journey that couldn't run. That's why neither review is optional.

{{< mermaid >}}
flowchart LR
  P[1. Propose<br/>opsx:propose] --> RP[2. Review plan<br/>ponytail-review]
  RP --> FP[3. Fix plan]
  FP --> A[4. Apply<br/>opsx:apply + checks + sonar]
  A --> RC[5. Review code<br/>ponytail-review]
  RC --> FC[6. Fix code]
  FC --> AR[7. Archive<br/>opsx:archive]
  AR --> R[8. Retrospective]
  R --> PR[9. Branch + PR]
  PR --> M[10. Merge on green]
  R -. lessons .-> P
{{< /mermaid >}}

## The whole thing

```markdown
## Change workflow

Every change goes through these steps in order. Run each step yourself, without
waiting to be asked. Stop only where a step needs the user's decision: a material
ambiguity while proposing, or a choice a review raises.

1. **Propose**: `/opsx:propose`. Done when every planning artifact exists and
   `openspec validate <change> --strict` passes.
2. **Review the plan**: `/ponytail:ponytail-review` on the change's artifacts,
   checked against the code they describe.
3. **Fix the plan**: apply every review point. Done when the artifacts reflect
   each one and strict validation passes again.
4. **Apply**: `/opsx:apply`. Done when every task is checked, and
   `pnpm test:coverage`, `pnpm tsc --noEmit`, `pnpm prettier --check .`,
   `pnpm lint`, `pnpm playwright test` and `sonar-check` all pass. Run
   `sonar-check` last, after `pnpm test:coverage`, so it reads fresh coverage.
   A failed SonarQube quality gate is a failed check: fix the issues it lists
   rather than silencing them with `// NOSONAR`.
5. **Review the code**: `/ponytail:ponytail-review` on the uncommitted diff and
   the code it touches.
6. **Fix the code**: apply every review point, and rerun the checks the fix affects.
7. **Archive**: `/opsx:archive`. It syncs the delta specs into `openspec/specs/`
   and moves the change into `openspec/changes/archive/`.
8. **Retrospective**: write `retrospective.md` in the archived change's folder,
   with three sections:
   - **Caught by review**: each finding from both reviews and how it was fixed.
   - **Surprises**: each failed check, flaky test, spec deviation or bug found
     during apply or archive, with its cause, or "cause unknown".
   - **Lessons**: what to do differently next time, each naming the step, file
     or habit it changes.

   Done when every review finding and every failure from this change has a line.
   Write "None" under a section with nothing to report.

9. **Branch and PR**: a new branch named after the change, with two commits.
   The first is the code, tests and docs. The second is `openspec/`,
   retrospective included, titled "Archive <change>: … joins the main specs".
   Push, then open the PR with the `pr` skill's template.
10. **Merge on green**: once CI passes, `gh pr merge <n> --rebase --delete-branch`,
    then pull `main`. Auto-merge is disabled on this repository, so check the CI
    status and merge by hand.

Both reviews have caught real bugs, such as a restore that could wipe all data
and an e2e journey that couldn't run, so neither is optional.
```

That's it. The rest of `CLAUDE.md` is a heading.

## Why it works

### Every step has a "done when"

An agent told to "propose a change" stops when it feels done. An agent told "done when every artifact exists and `openspec validate --strict` passes" stops when a command exits 0. Every step names its exit condition. The only judgment calls left are the ones I want: ambiguity in the plan, or a choice a review raises.

### Run it yourself, stop only for decisions

The first paragraph does the heavy lifting. Without it, the agent finishes a step and asks "shall I continue?". With it, I give it a feature and come back to a merged PR, or to a single question.

### The project context lives in `openspec/config.yaml`, not `CLAUDE.md`

`CLAUDE.md` is the process. The domain lives in OpenSpec's config, which is injected when the agent writes artifacts:

```yaml
schema: spec-driven
context: |
  Strength v2: a local-first, installable PWA for logging barbell strength
  training. ...
  Source documents (read the relevant ones before writing artifacts):
  - docs/PRD.md: goals, journeys J1-J5, requirement IDs ...
  - docs/domain-model.md: entities, program type rules, invariants (§9)
  ...
  Key domain rules:
  - Finishing a session is the only automatic change to progression state.
    On a miss the weight holds; the app never lowers weight automatically.
  ...
rules:
  proposal:
    - Cite the PRD requirement IDs and journeys the change delivers.
    - Include a Non-goals section.
  specs:
    - Write scenarios in the lifter's terms with concrete numbers and units,
      e.g. "3x5 @ 100 kg".
    - Cover the miss, no-baseline and both-units cases where they apply.
  tasks:
    - Write tests first - Vitest for domain and storage, Serenity/JS
      Screenplay for each user journey touched.
    - Keep each task small enough to finish and verify on its own.
```

The per-artifact `rules` are where quality comes from. "Concrete numbers and units" makes specs testable. "Tests first" plus "each task verifiable on its own" makes `tasks.md` read like this:

```markdown
- [x] 1.1 Write failing tests in `src/domain/history.test.ts` for
  `trainingWeeks(startedAts, today)` ... Set `process.env.TZ = 'Europe/London'`
  at the top of the file so the 23:30 and daylight-saving tests fail on a UTC
  machine when the code is wrong. Verify they fail with
  `pnpm vitest run src/domain/history.test.ts`.
- [x] 1.2 Implement `trainingWeeks` in `src/domain/history.ts` as design.md
  describes. Verify the tests from 1.1 pass.
```

### Two reviews, same tool

ponytail is a "lazy senior dev" skill: the smallest change that fully solves the problem. Its review asks whether each line is needed, whether the logic is right, and whether risky code has a test. That's the right lens both before and after code exists.

The plan review is the cheaper one, and it pays most. From the retrospectives:

- *"The time-zone tests could pass on a UTC machine."* Under UTC, stepping by 24 h gives the same answer as local dates, so the daylight-saving tests checked nothing. Fixed by pinning `TZ` in the test file, before any code was written.
- *"The round-trip journey couldn't prove anything."* The fixture already started the program the import was supposed to create, so the e2e test would pass with the feature missing.
- *"The e2e journey would fail across midnight."* Two workouts on the real clock could land on two days. Fixed with a fake clock.

None of these are style nits. Each is a test that would have gone green while proving nothing.

The code review often finds nothing, and says what it checked: Monday maths on Sunday and Monday, future-dated sessions, that the pinned `TZ` can't leak into other test files. "Nothing to fix" with evidence is a fine result.

### SonarQube as a check, not a dashboard

SonarQube runs on the dev VM (see [one VM instead of seven devcontainers]( {{< ref "post/coding/2026/one-vm-instead-of-seven-devcontainers/" >}} )), not in CI. A small `sonar-check` script wraps `sonar-scanner` with `-Dsonar.qualitygate.wait=true`, and on failure prints the failed gate conditions and the new-code issues as `file:line rule message`:

```bash
if [ "$status" -ne 0 ]; then
  echo "--- failed quality gate conditions ---"
  curl -fsSu "$token:" "$host/api/qualitygates/project_status?projectKey=$key" \
    | jq -r '.projectStatus.conditions[]? | select(.status == "ERROR")
             | "\(.metricKey): \(.actualValue) (fails when \(.comparator) \(.errorThreshold))"'
  echo "--- open issues in new code (file:line rule message) ---"
  curl -fsSu "$token:" "$host/api/issues/search?components=$key&resolved=false&inNewCodePeriod=true&ps=100" \
    | jq -r '.issues[] | "\(.component | sub("^[^:]*:"; "")):\(.line // "-")  \(.rule)  \(.message)"'
fi
```

That output is the point. The agent doesn't need a browser or a dashboard. It gets a list it can fix.

Two rules in `CLAUDE.md` make it stick:

- **Run it last, after `pnpm test:coverage`**, so the coverage it reads is fresh.
- **No `// NOSONAR`.** A failed gate is a failed check. Without this line, suppressing the warning is the shortest path to green, and the agent will find it.

The one exception went through me: Sonar's S2699 ("test has no assertion") doesn't recognise Serenity/JS's `Ensure.that`, so `sonar-project.properties` ignores it under `e2e/**`, with a comment saying why. Ignoring a rule is a decision for the human, not the agent.

{{< notice type="tip" >}}
Only new code fails the gate. The repo had 62 old cases of S6759 (read-only props) when the gate went in. They don't block anything; every new component just has to get it right.
{{< /notice >}}

### The retrospective closes the loop

The retrospective has three fixed sections and a hard rule: every review finding and every failed check gets a line. "Cause unknown" is allowed. Silence isn't.

The `Lessons` section has to name the step, file or habit it changes. That turns "be more careful" into something actionable:

> **Propose, local-date logic:** when a change depends on local dates, plan a pinned `TZ` in its tests from the start. This machine runs AEDT and CI runs UTC, so neither shows a time-zone bug by default.

> **Propose, dates in tasks:** check any calendar fact (a daylight-saving date, a weekday) against a calendar or `date` before writing it into a task.

It also shows when a lesson didn't land. Sonar's read-only props rule failed the gate in one change; the retro said "type props as `Readonly<…>` from the start". The next change failed on the same rule, on an existing component whose props were touched. The next retro sharpened it: check every *changed* component signature, not just new ones.

The retros live in `openspec/changes/archive/`, next to the proposal they belong to.

### Two commits per PR

Code, tests and docs in commit one. `openspec/` (spec sync, archived change, retrospective) in commit two, titled "Archive <change>: … joins the main specs". The history reads as pairs:

```text
d61b244 Consistency strip: History shows recent training days (HS-4)
b81fbf8 Archive m4-consistency-strip: recent training days join the main specs
8c62400 Routine files: export a routine, and import one (CT-3)
b067a1b Archive m4-routine-files: routine import and export join the main specs
```

The code commit is reviewable on its own. The archive commit is noise you can skip.

## How it grew

`CLAUDE.md` started as 17 lines: propose, review, apply, review, archive, PR. Then each addition came from something that went wrong, or something I kept doing by hand:

1. **Retrospective** — I kept re-explaining the same lessons.
2. **SonarQube gate** — issues were reaching review that a scanner catches for free.
3. **oxlint and Renovate** — lint in the check list; dependencies kept current without me.

Each was one commit titled `Workflow: …`. The process changes the same way the code does.

## What I'd watch

- **The plan review is where the value is.** If you only keep one review, keep that one. A test that proves nothing is cheap to fix in `tasks.md` and expensive to find later.
- **Write the "done when" as a command.** Anything softer, and the agent decides.
- **Keep the domain out of `CLAUDE.md`.** Put it in `openspec/config.yaml` where it's injected only when artifacts are written.
- **Tool availability matters.** One change ran both reviews by following the ponytail checklist by hand, because the VM's `claude` CLI wasn't logged in. The retro caught it. A process that quietly degrades is still better than one that quietly skips.
