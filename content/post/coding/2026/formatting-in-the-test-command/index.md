---
title: "Formatting Belongs in the Test Command, Not Just the Commit Hook"
date: 2026-11-22T14:23:53+11:00
publishDate: 2026-11-22
draft: true
layout: "post"
tags: ["formatting", "linting", "spotless", "prettier", "eslint", "gradle", "maven", "pnpm", "git-hooks", "claude-code", "ai"]
categories: ["coding"]
description: "Why I moved formatting and lint fixes out of the commit hook and into each project's normal test command, so coding agents get formatted, lint-clean code without being told to."
author: "Paul"
---

Until recently my projects formatted code in one place: a pre-commit hook. Write some code, commit, and the hook tidies it up on the way in. That works fine when a person is doing the work.

It works less well when an agent is. Most of the code in my projects is now written by Claude Code, and the agent's loop isn't "write, commit". It's "write, run the tests, run `sonar-check`, report done". The commit comes later.

So I've changed the rule. Formatting and lint fixes now run inside each project's normal test command. The agent runs the tests it was going to run anyway, and the code comes out formatted and lint-clean. Nobody has to remember a separate step.

## What was wrong with commit-only

With formatting only at commit, the order of events was off:

1. The agent writes code.
2. It runs the tests and `sonar-check` on that code, unformatted.
3. It reports done.
4. Later, at commit, the hook rewrites the files.

So the tests and Sonar checked code that the hook then changed. Formatting is meant to be harmless, but "the code I tested isn't quite the code I committed" isn't a nice place to be. Worse, lint is more than formatting. A lint error that `--fix` can't repair only turned up at commit, after the work had already been reported done.

The workaround was to tell the agent to format. My [quality gates post](/pauls-blog/post/coding/2026/ai-code-quality-gates-java-kotlin/) says it plainly: "`spotlessCheck` in the build, `spotlessApply` in the agent's instructions". That's one more instruction for the agent to follow, and one more it can skip. If the build does it, there's nothing to skip.

## The rule now

All my projects live in a VM, and the VM writes a `~/projects/CLAUDE.md` that every Claude Code session under `~/projects` reads. The relevant part:

> Formatting and linting are part of the build, so the normal test command does them with no separate step: it formats and auto-fixes first, then fails on what is left. In CI (`CI` is set) it only checks, never rewrites.

There are three parts to that, and each matters:

- **Fix first, then fail.** Locally, the test command formats and applies every auto-fix, then fails on whatever is left. What's left is a real problem the agent has to look at, not whitespace.
- **CI only checks.** If CI formatted the code, it would pass, and the fix would never make it back into the repo. So with `CI` set, the same command checks and fails instead. GitHub Actions sets `CI` by default, so there's nothing to configure.
- **The commit hook stays, as a backstop.** It catches edits that never went through the build, like a quick fix in the IDE.

{{< mermaid >}}
flowchart LR
    A[test command] --> B{CI set?}
    B -- no --> C[format + auto-fix] --> D[lint check] --> E[tests]
    B -- yes --> F[format check + lint check] --> E
    G[git commit] --> H[hook: format staged files<br/>backstop]
{{< /mermaid >}}

## JavaScript and TypeScript

The `test` script does the fixing itself:

```json
"lint": "eslint . && prettier --check .",
"format": "eslint --fix . && prettier --write .",
"fix": "if [ -n \"$CI\" ]; then pnpm lint; else pnpm format; fi",
"test": "pnpm fix && vitest run"
```

`pnpm test` formats and fixes locally, and only checks in CI. CI can also call `pnpm lint` on its own.

I deliberately didn't use a `pretest` script. It would look neater, but pnpm may not run pre and post scripts, and a formatting step that silently doesn't run is worse than none. So the fix is part of `test` itself.

Some projects use oxlint instead of ESLint. Same scripts, with `oxlint` swapped in for `eslint`.

## Gradle

Spotless, PMD and SpotBugs all hook their checks into `check`, so `./gradlew build` already fails on unformatted code. The missing piece was making it fix the formatting first, but only outside CI:

```kotlin
// Format before compiling, locally only: CI must fail on unformatted code, not fix it.
if (System.getenv("CI") == null) {
    tasks.named("compileJava") { dependsOn("spotlessApply") }
}
```

Hooking it to `compileJava` means it runs for anything that compiles: `test`, `build`, running the app. The agent can't get to the tests without the code being formatted first.

## Maven

Maven gets the same split with a profile that's only active when `CI` isn't set:

```xml
<profile>
  <id>format</id>
  <activation><property><name>!env.CI</name></property></activation>
  <build><plugins><plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <executions><execution>
      <id>apply</id><phase>process-sources</phase><goals><goal>apply</goal></goals>
    </execution></executions>
  </plugin></plugins></build>
</profile>
```

`spotless:apply` runs at `process-sources`, early in every build. The main build still binds `spotless:check`, `pmd:check` and `spotbugs:check` to `verify`, so CI checks everything without changing anything.

## The hook that's left

The pre-commit hook didn't go away. It just isn't doing the main job any more. Where there's a `package.json` it's Husky with lint-staged. In a JVM-only repo it's a native git hook in `.githooks/`, because adding a `package.json` just to get Husky would make Node a requirement for committing to a Java project.

One detail in the lint-staged config is worth sharing. Each file has to match only one glob, because lint-staged runs globs in parallel, and two tools rewriting the same file at the same time race each other:

```js
const code = '*.{js,jsx,ts,tsx,mjs,cjs}';
export default {
  [code]: ['eslint --fix', 'prettier --write'],
  [`!(${code})`]: 'prettier --write --ignore-unknown',
};
```

## Rolling it out

Writing a rule in `CLAUDE.md` doesn't change any existing repo's build. So the VM also installs an `apply-project-standards` skill. In a Claude Code session I say "apply project standards", and it:

- runs `standards-check`, a read-only script that lists one gap per line
- fixes each gap from a set recipe (the Gradle and Maven setup above, the `package.json` scripts, the hooks)
- gets the build green after each part

The last check is the important one. The skill runs the test command again with `CI=true` and checks that `git status` is still clean afterwards. That proves both halves at once: the local run left the code formatted, and the CI run checks without rewriting anything.

New checkers usually find problems in code that was already there. When there are only a few, the skill fixes them. When there are many, it stops and asks me what to do. The one thing it must never do is switch a checker off to get the build green.

## The result

The agent doesn't need an instruction to format, or to run a linter, or to remember a fix command. It runs the tests, as it always did, and formatting and lint fixes come along with them. By the time it says "done", the code it tested is the code that will be committed, and any lint error that needs a human has already shown up.
