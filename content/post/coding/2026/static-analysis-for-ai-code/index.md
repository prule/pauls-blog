---
title: "Static Analysis as the Agent's Feedback Loop"
date: 2026-11-30T15:14:37+11:00
publishDate: 2026-11-30
draft: true
layout: "post"
tags: ["ai", "claude-code", "static-analysis", "sonarqube", "pmd", "eslint", "oxlint", "playwright", "code-coverage", "testing"]
categories: ["coding"]
description: "How I stack linters, type checks, PMD, coverage, Playwright and SonarQube into one definition of done that a coding agent runs itself, and the few rules that make an agent act on what they find."
author: "Paul"
---

A coding agent will tell you it's done when the code is written. What I want "done" to mean is that the code passed every check I'd have run myself. The way to get that isn't a better prompt. It's tools that fail, run by the agent, with output it can act on.

So every project I work on has a stack of them: a formatter and linter, the type checker, unit tests with coverage, Playwright end-to-end tests, PMD and SpotBugs in the Java projects, and SonarQube over the top. The agent runs them all before it reports back. Here's how they fit together, and the handful of rules that make them work with an agent rather than just next to one.

I've written about parts of this before: a [deep dive on Java and Kotlin gates](/pauls-blog/post/coding/2026/ai-code-quality-gates-java-kotlin/), [`sonar-check`](/pauls-blog/post/coding/2026/sonar-check-for-agents/), and [moving formatting into the test command](/pauls-blog/post/coding/2026/formatting-in-the-test-command/). This post is the overview: what each layer is for, and how they combine.

## A real definition of done

Here's the line from my strength2 project's `CLAUDE.md`, a React and TypeScript app built change by change with Claude Code:

> Done when every task is checked, and `pnpm test:coverage`, `pnpm tsc --noEmit`, `pnpm lint` (oxlint and Prettier), `pnpm tokens:check`, `pnpm playwright test` and `sonar-check` all pass. Run `sonar-check` last, after `pnpm test:coverage`, so it reads fresh coverage. A failed SonarQube quality gate is a failed check: fix the issues it lists rather than silencing them with `// NOSONAR`.

Every item is a command that exits non-zero when something's wrong. There's nothing in there like "make sure the code is clean". An agent can't check a feeling. It can check an exit code.

## The layers, fastest first

Each layer catches a different kind of mistake, and they're ordered so the cheap ones fail first.

{{< mermaid >}}
flowchart LR
    A[format + lint<br/>seconds] --> B[type check<br/>seconds]
    B --> C[unit tests + coverage<br/>seconds to a minute]
    C --> D[Playwright e2e<br/>minutes]
    D --> E[SonarQube gate<br/>30s to minutes]
{{< /mermaid >}}

**Formatter and linter.** Prettier with ESLint or oxlint in TypeScript projects; Spotless in Java. They catch the small stuff: unused variables, missing `await`s, inconsistent style. The test command fixes what it can first and fails on the rest, so the agent never has to remember to format, and what's left is something it has to actually look at.

**Type checker.** `tsc --noEmit` with `"strict": true`. Agents are good at writing code that looks right, and strict types catch a lot of the code that looks right but isn't: a value that can be `undefined`, a field that doesn't exist on that type, a function called with its arguments in the wrong order.

**PMD and SpotBugs (Java).** The Java equivalent of a strict linter, run in the Gradle or Maven build so they fail it. PMD reads the source, SpotBugs reads the bytecode, and between them they find likely bugs: unclosed resources, ignored return values, possible null dereferences, needless complexity.

**Unit tests with coverage.** Tests prove the behaviour; coverage shows what the tests never touched. I don't set a global coverage percentage. Instead the coverage report (`lcov` from Vitest, JaCoCo from Gradle) goes to SonarQube, which judges coverage on *new code*. That's the right question for an agent: did the code you just wrote get tested? An old untested corner of the codebase shouldn't fail today's change, and new untested code should.

**Playwright end-to-end tests.** Unit tests can all pass while the app doesn't work: a route isn't wired up, a button is hidden, a form never submits. Playwright drives a real browser through real journeys and catches that. Several of my projects write these in the Screenplay pattern with Serenity/JS, so a test reads as an actor doing tasks rather than a list of clicks and selectors.

**SonarQube.** Last, because it's the slowest and because it needs the coverage report from the test run. It brings its own large rule set (bugs, security hotspots, code smells, duplication), and its quality gate pulls it together with the coverage figure into one pass or fail for the new code.

There's also a project-specific check in that list: `pnpm tokens:check`, which fails if the generated design tokens are out of date with their source. A check like that is cheap to write and turns a project-specific rule into one more exit code.

## What makes the output useful to an agent

Running the tools is the easy part. Four rules decide whether the agent fixes what they find.

**1. Fail, don't report.** A dashboard or an HTML report is something a person might read later. An agent works from what's in its terminal. Every tool here is set up to fail the command, so a problem stops the agent until it's fixed. SonarQube is the one that needed work, because on its own it just uploads an analysis. `sonar-check` waits for the quality gate and fails the command when the gate fails.

**2. Print `file:line`.** When a check fails, the output should say where. Linters, compilers and test runners already do. For SonarQube, `sonar-check` fetches the failing issues from its API and prints each one as `file:line rule message`. The agent opens the file, goes to the line, and fixes it, without opening a browser.

**3. Suppressing isn't fixing.** The quickest way for an agent to make a finding go away is `// NOSONAR`, `// eslint-disable-next-line` or `@SuppressWarnings`. So the instructions say up front that a suppression isn't a fix, and my pipeline's QA agent treats a suppression without a written reason as a defect in its own right. Suppressions do have a place, though. strength2's SonarQube config has one, with a comment saying why:

```properties
# Screenplay journeys assert with Serenity's Ensure.that, which S2699 doesn't recognise as an assertion.
sonar.issue.ignore.multicriteria=e1
sonar.issue.ignore.multicriteria.e1.ruleKey=typescript:S2699
sonar.issue.ignore.multicriteria.e1.resourceKey=e2e/**
```

That's a rule that doesn't understand a library, scoped to the folder where that library is used. A suppression like that is a decision. A bare `// NOSONAR` on the line that failed is a dodge.

**4. Put it where the agent already looks.** The rules live in `CLAUDE.md`, which every Claude Code session reads. My dev VM writes the shared standards to `~/projects/CLAUDE.md` for every project, starting with:

> Before reporting work as done, run the project's tests and `sonar-check`, and fix what fails.

And where a project uses my sdlc-pipeline, the same checks are the pipeline's **Verify** command. The implementer, QA and senior dev agents each have to run it and paste the real output. A claim that the tests pass isn't evidence; the output is.

## Order matters

A few rules about ordering, most of them written into my pipeline's QA agent:

- **Cheap checks first.** There's no point waiting minutes for Playwright or SonarQube on code that doesn't type-check.
- **SonarQube after coverage.** Sonar reads the coverage file the test run wrote. Run it first and it judges stale coverage, or none at all.
- **Static analysis only on code that builds.** My QA agent skips the SonarQube scan when Verify fails, and says so. A scan of broken code is noise.
- **A scan that can't run isn't a finding.** If the SonarQube server is down or there's no token, that's an infrastructure problem. The agent reports it once and judges the change on the rest of the evidence, instead of retrying in a loop or "fixing" code that was fine.

## What the tools don't catch

None of this checks that the code does what was asked. A change can pass every gate and still solve the wrong problem. That's what the spec and the reviews are for, and in strength2 the reviews have caught real bugs, such as a restore that could wipe all data. The tools make sure the code is well-formed, typed, tested and free of known bad patterns. That frees the reviews, mine and the agents', to look at whether it's right.

The tools have a cost, too. A full run takes minutes, and the first time a new checker runs on an existing codebase it usually finds a pile of old problems. My `apply-project-standards` skill handles that by stopping and asking when a new checker finds many existing violations: fix them now, or narrow the rules and fix them later. The one thing it must never do is switch a checker off to get green.

## The short version

- Make every check a command that fails.
- Put them in one definition of done, fastest first, in the file the agent reads.
- Make failures print `file:line`.
- Treat a suppression as a decision that needs a reason, not a fix.
- Let the agent run it all, and ask for the output, not the claim.
