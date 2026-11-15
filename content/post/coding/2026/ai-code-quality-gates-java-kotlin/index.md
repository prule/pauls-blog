---
title: "Quality Gates for AI-Generated Java and Kotlin: Beyond SonarQube"
date: 2026-11-15T12:36:53+11:00
publishDate: 2026-11-15
draft: false
layout: "post"
tags: ["java", "kotlin", "gradle", "ai", "static-analysis", "archunit", "nullaway", "error-prone", "pitest", "postgres", "claude-code"]
categories: ["coding"]
description: "Which static analysis and test tools catch the mistakes AI coding agents make in Java, Kotlin and SQL, how to wire them into the agent's own build loop, and a demo where 11 typical AI defects pass a well-reviewed build and every one fails a gate."
author: "Paul"
---

SonarQube alone won't stop an AI agent from shipping bad code. Sonar *reports*. Agents don't read dashboards. What works: a small set of tools that **fail the build**, wired into the one command the agent already runs, with the config files locked so the agent can't switch them off.

I added these gates to a small Spring Boot project that an agent had built, item by item, with three AI reviews per change. Then I seeded it with 11 typical AI mistakes. **All 11 passed the original build. All 11 failed the new one.** Adding the gates also turned up a latent NPE and an untested boundary in code that had passed every review.

The short list, in order of value for AI-generated code:

1. **ArchUnit** (or Konsist): enforces your architecture as tests
2. **Error Prone + NullAway with JSpecify**: compile-time bug and null checks (Java)
3. **PIT mutation testing**: catches tests that assert nothing
4. **Dependency rules** (Maven Enforcer, or Gradle resolution strategy): blocks banned and drifting dependencies
5. **Testcontainers + Flyway validate + Squawk**: real database, schema checks, migration safety
6. **detekt** and Kotlin `allWarningsAsErrors` (Kotlin)
7. **Spotless**, **OpenRewrite**, **Gitleaks**, **OWASP dependency-check**: hygiene

Skip **Checkstyle** and **PMD** if Sonar is already gating. They mostly duplicate it.

{{< notice type="tip" >}}
The rule of thumb: if a tool's output is a red line in `./gradlew build`, the agent fixes it in seconds. If it's a web report, nobody fixes it.
{{< /notice >}}

## The evidence

The project: a stateless Spring Boot 4 / Java 25 REST API for exact decimal arithmetic, about 1,600 lines. It already had a decent build: JUnit, ArchUnit layer rules, Spotless. Every change had gone through OpenSpec, three AI code reviews and CI. The code, the gates and the demo are public: [math-api-outcome-based-backlog-openspec-quality-gates](https://github.com/prule/math-api-outcome-based-backlog-openspec-quality-gates).

### Seeded defects

Each defect is a script that writes a realistic AI mistake into a copy of the project, usually with a test that passes. A runner builds every defect twice: once against the project before the gates (a git tag, `baseline`) and once against the gated tree.

| # | Defect | Baseline | With gates | Caught by |
|---|--------|----------|------------|-----------|
| 01 | `value.equals(BigDecimal.ZERO)`: `0.00` is not "zero" | ✅ passes | ❌ fails | Error Prone `BigDecimalEquals` |
| 02 | `new BigDecimal(0.1)` | ✅ passes | ❌ fails | Error Prone `BigDecimalLiteralDouble` |
| 03 | `value.setScale(2, HALF_EVEN); return value;` | ✅ passes | ❌ fails | Error Prone `ReturnValueIgnored` |
| 04 | Parser returns `null` on bad input | ✅ passes | ❌ fails | NullAway |
| 05 | Same, plus `@SuppressWarnings("NullAway")` | ✅ passes | ❌ fails | no-suppressions check |
| 06 | Average via `doubleValue()` | ✅ passes | ❌ fails | ArchUnit: no floating point |
| 07 | `System.out.println` left in | ✅ passes | ❌ fails | ArchUnit: no standard streams |
| 08 | `@Autowired` private field | ✅ passes | ❌ fails | ArchUnit: no field injection |
| 09 | `junit:junit:4.13.2` added | ✅ passes | ❌ fails | Banned dependency |
| 10 | `commons-lang3:3.+` | ✅ passes | ❌ fails | Dynamic version |
| 11 | Clamp test, 100% line coverage, only `isNotNull()` | ✅ passes | ❌ fails | PIT: score 90, threshold 95 |

The whole run takes about 90 seconds. Defect 05 matters most: it's what an agent does when a gate gets in its way, and the gate for it is a 15-line Gradle task.

### Real findings in reviewed code

Seeded bugs prove the gates work. These two prove they're worth it:

- **A latent NPE.** The standards say adding a unary operation needs only a new enum constant. The controller called `calculation.b().toPlainString()`, and `b` is null for a unary operation. The first unary operation would have returned a 500. NullAway flagged it as soon as `b` was marked `@Nullable`. Three AI reviews and the human who wrote the standards didn't.
- **An untested boundary.** PIT showed the request-size check `body.length > MAX_BYTES` could become `>=` with no test failing. The fix is a test that sends exactly 16,384 four-byte (UTF-32) characters, the one body that hits the byte cap exactly. The mutation score went from 97% to 100%.

## Why AI code needs different gates

AI agents fail in predictable ways. Sonar catches the generic ones (complexity, duplication, smells). These it mostly misses:

| Failure mode | Tool that catches it |
|---|---|
| Controller calls the repository directly; field injection; logic in the wrong package | ArchUnit |
| Returns `null` from a method callers treat as non-null | NullAway |
| Ignores a return value, misuses `Optional` or `CompletableFuture`, wrong `equals` | Error Prone |
| Tests that run every line and assert nothing useful. 100% coverage, zero protection | PIT |
| Adds JUnit 4 next to JUnit 5, uses a dynamic version, invents a package name | Dependency rules, OWASP |
| Silences a check instead of fixing the code | No-suppressions check + locked config |
| SQL that passes on H2 and fails on Postgres | Testcontainers |
| `ADD COLUMN ... NOT NULL DEFAULT` that locks a large table in production | Squawk |
| An API key pasted into a test fixture | Gitleaks |

## Integrating it into an AI build

Adding the tools isn't enough. The agent has to hit them **while it works**, not in CI an hour later.

{{< mermaid >}}
flowchart LR
  A[Agent edits code<br/>runs build itself] --> B{Stop hook<br/>fast gate}
  B -- red --> H[Human told<br/>decides on retry]
  H -- fix it --> A
  B -- green --> C[Agent says done]
  C --> D{CI gate<br/>full suite}
  D -- fail --> A
  D -- pass --> E[Human review]
  P[PreToolUse hook] -. blocks edits to .-> G[(Gate config)]
{{< /mermaid >}}

### 1. One command, two tiers

Everything hangs off `./gradlew build` (or `mvn verify`). No separate scripts, no "also run X".

- **Fast** (every agent turn): compile with Error Prone + NullAway, Spotless, unit and slice tests, ArchUnit, dependency rules, no-suppressions. About 10 seconds on the demo project.
- **Slow** (CI, and before the agent opens a PR): `./gradlew pitest`, integration tests, OWASP, Sonar. PIT took 20 seconds here; on a real codebase it's minutes.

### 2. Put the gates in a file the agent can't touch

The agent legitimately edits `build.gradle.kts` to add dependencies, so you can't lock that file. Move the gates out of it. In Gradle, use a precompiled script plugin in `buildSrc`:

```kotlin
// build.gradle.kts: the agent may edit this, except the gates line
plugins {
    java
    id("org.springframework.boot") version "4.1.2"
    id("com.diffplug.spotless") version "8.11.0"
    `quality-gates` // buildSrc/src/main/kotlin/quality-gates.gradle.kts
}
```

```kotlin
// buildSrc/build.gradle.kts
plugins { `kotlin-dsl` }
repositories { gradlePluginPortal() }
dependencies {
    implementation("net.ltgt.errorprone:net.ltgt.errorprone.gradle.plugin:5.2.0")
    implementation("info.solidsoft.pitest:info.solidsoft.pitest.gradle.plugin:1.19.0")
}
```

In Maven, do the same with a parent POM the service doesn't own.

### 3. Tell the agent, in its instructions file

```markdown
Quality gates (docs/STANDARDS.md §9) run in `./gradlew build`; PIT runs as `./gradlew pitest` before every PR.
Fix a failing gate in the code. Never suppress, disable or loosen a gate, and never edit `buildSrc/`,
`ArchitectureTest`, an existing `package-info.java`, `.claude/` or `.github/`: hooks block it. If a rule looks wrong,
stop and ask the user.
```

### 4. Enforce it with hooks

Instructions are advice. Hooks are enforced. In Claude Code:

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Edit|Write|MultiEdit",
        "hooks": [{ "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-quality-gates.sh" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/verify.sh", "timeout": 600 }] }
    ]
  }
}
```

The **Stop hook** runs when the agent finishes a turn. It can *block* the stop (exit 2 sends the agent back to work) or *inform* the human. I started with blocking and switched to informing:

```bash
#!/usr/bin/env bash
# .claude/hooks/verify.sh: tells the human when a turn ends on a red build. Never blocks, so it cannot loop.
# Only when code or build files changed; a docs-only turn need not wait for a build.
git status --porcelain -- src buildSrc build.gradle.kts settings.gradle.kts | grep -q . || exit 0
out=$(./gradlew build --console=plain 2>&1) && exit 0
# The lines that name what failed: compiler findings, failed tests, Gradle's own reason.
failures=$(grep -v '^OpenJDK\|INFO\|^> Task\|^> Run with\|^> Compilation failed\|^> There were failing tests\|^> Get more help' <<<"$out" |
  grep -E '(error|warning): \[|FAILED$|^> |Suppressions are not allowed|Banned dependency' |
  sed -E "s#^[ >]*##; s#$PWD/##" | awk '!seen[$0]++' | head -10)
jq -n --arg failures "$failures" '{
  systemMessage: ("⚠️ ./gradlew build is red after this turn. Reply \"fix the build\" to let the agent try again.\n" + $failures)
}'
```

On a red build, the human sees something like:

```text
⚠️ ./gradlew build is red after this turn. Reply "fix the build" to let the agent try again.
ArchitectureTest > mainClassesUseNoFloatingPoint() FAILED
```

Why not block? A blocking Stop hook is a retry loop with no human in it:

- **It can spin.** Without a `stop_hook_active` check, a build the agent can't fix (a flaky test, a broken environment) traps it until someone presses Esc. Many hook examples leave the check out. With it, you get one forced retry, but even that single retry is a full agent turn with no limit on tool calls.
- **Pressure produces bad fixes.** An agent forced to go green will reach for whatever turns the build green. The PreToolUse hook stops it loosening the gates, but not deleting the failing test or weakening an assertion.
- **The agent already runs the build.** The instructions file tells it to run `./gradlew build` before saying it's done. The hook is a safety net for when it forgets or gives up, and at that point a human should decide.

Your reply ("fix the build") is the approval. A Stop hook can't show you a prompt itself; it can only block or allow.

If you want more autonomy, there are two good middle grounds. Count retries per `session_id` in a state file. Or stop retrying when the failure signature (failed tests plus first error lines) is the same as last time. Either way, also add a PreToolUse hook that returns `permissionDecision: "ask"` for edits to existing test files. That puts the human approval on the risky *fix* rather than on the retry.

One more detail from practice: **skip the build when nothing changed.** Otherwise every "here's my plan" turn waits for a build.

The **PreToolUse hook** locks the gates:

```bash
#!/usr/bin/env bash
# .claude/hooks/protect-quality-gates.sh
input=$(cat)
path=$(jq -r '.tool_input.file_path // empty' <<<"$input")
rel=${path#"$CLAUDE_PROJECT_DIR"/}
block() { echo "$rel is a quality gate and is human-owned. Fix the code instead; if the rule is wrong, stop and ask the user." >&2; exit 2; }
case "$rel" in
  buildSrc/*|.claude/hooks/*|.claude/settings.json|.github/workflows/*|.github/CODEOWNERS) block ;;
  src/test/java/com/acme/ArchitectureTest.java) block ;;
  # Removing @NullMarked would switch NullAway off for the package; new packages may add one.
  src/main/java/*/package-info.java) [ -e "$path" ] && block ;;
  build.gradle.kts)
    # Dependencies may change here; the line that applies the gates may not.
    jq -r '.tool_input.old_string // "", .tool_input.content // ""' <<<"$input" | grep -q 'quality-gates' &&
      ! jq -r '.tool_input.new_string // .tool_input.content // ""' <<<"$input" | grep -q '`quality-gates`' && block ;;
esac
exit 0
```

`package-info.java` is the easy one to miss. Deleting `@NullMarked` switches NullAway off for a whole package, and the diff looks harmless.

Back the hook with `CODEOWNERS` on the same paths, so a human approves any rule change in the PR.

### 5. Ban suppressions in the build

The hook stops the agent from loosening the config. It doesn't stop `@SuppressWarnings("NullAway")` in the source. A small task does:

```kotlin
val noSuppressions by tasks.registering {
    val sources = fileTree("src") { include("**/*.java") }
    val root = projectDir
    inputs.files(sources)
    doLast {
        // Error Prone and NullAway check names are CamelCase; javac lint names such as "unchecked" stay allowed.
        val banned = Regex("""@SuppressWarnings\([^)]*"[A-Z]|@Disabled\b|@ArchIgnore\b""")
        val hits = sources.files.flatMap { file ->
            file.readLines().mapIndexedNotNull { i, line ->
                if (banned.containsMatchIn(line)) "${file.relativeTo(root)}:${i + 1}: ${line.trim()}" else null
            }
        }
        if (hits.isNotEmpty()) throw GradleException("Suppressions are not allowed:\n" + hits.joinToString("\n"))
    }
}
tasks.named("check") { dependsOn(noSuppressions) }
```

My first version banned every `@SuppressWarnings` and immediately failed on a legitimate `"unchecked"` cast. The CamelCase trick bans only the analyser checks.

### 6. Strict on new code, baselines for old code

Turning everything on in a legacy repo produces 4,000 errors and an agent that rewrites half the codebase. Instead:

- ArchUnit: `FreezingArchRule.freeze(rule)` records existing violations and fails only on new ones
- detekt: `detektBaseline`
- NullAway: `OnlyNullMarked=true`, then add `@NullMarked` one package at a time
- Sonar: gate on "new code" only
- PIT: on PRs, mutate only changed code (`scmMutationCoverage` in Maven)

On the 1,600-line demo project I turned everything on at once. That cost 15 NullAway findings and one Error Prone false positive, about half an hour of work.

## Setup, tool by tool

### Error Prone + NullAway + JSpecify

Gradle, in the `buildSrc` plugin:

```kotlin
import net.ltgt.gradle.errorprone.errorprone

plugins {
    java
    id("net.ltgt.errorprone")
    id("info.solidsoft.pitest")
}

dependencies {
    errorprone("com.google.errorprone:error_prone_core:2.52.0")
    errorprone("com.uber.nullaway:nullaway:0.14.3")
}

tasks.withType<JavaCompile>().configureEach {
    // Error Prone warnings fail the build too; a warning nobody must fix gets ignored.
    options.compilerArgs.add("-Werror")
    options.errorprone {
        disableWarningsInGeneratedCode = true
        error("NullAway")
        option("NullAway:OnlyNullMarked", "true")
        option("NullAway:JSpecifyMode", "true")
        // Tests pass null on purpose to prove the API rejects it.
        if (name == "compileTestJava") disable("NullAway")
    }
}
```

Maven:

```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <release>21</release>
    <compilerArgs>
      <arg>-Werror</arg>
      <arg>-XDcompilePolicy=simple</arg>
      <arg>--should-stop=ifError=FLOW</arg>
      <arg>-Xplugin:ErrorProne -XepDisableWarningsInGeneratedCode -Xep:NullAway:ERROR -XepOpt:NullAway:OnlyNullMarked=true -XepOpt:NullAway:JSpecifyMode=true</arg>
    </compilerArgs>
    <annotationProcessorPaths>
      <path><groupId>com.google.errorprone</groupId><artifactId>error_prone_core</artifactId><version>${errorprone.version}</version></path>
      <path><groupId>com.uber.nullaway</groupId><artifactId>nullaway</artifactId><version>${nullaway.version}</version></path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

Maven also needs `.mvn/jvm.config` with the `--add-exports jdk.compiler/...` lines from the Error Prone install docs.

Then add `org.jspecify:jspecify` as a dependency, and a `package-info.java` for each package:

```java
@NullMarked
package com.acme.orders;

import org.jspecify.annotations.NullMarked;
```

What I learned:

- **`-Werror` is not optional.** `BigDecimalEquals` and `BigDecimalLiteralDouble`, two of the most relevant checks for a money or maths codebase, are *warnings* by default. Without `-Werror`, defects 01 and 02 compile and ship.
- **Most NullAway findings are honest annotations, not bugs.** Of the 15, most were `Throwable.getMessage()` and `Map.get()`, which really are nullable. The fix is `@Nullable` on a parameter, or `Objects.requireNonNull` with a message. One finding was the real NPE above. That ratio is still worth it.
- **Your ArchUnit rules will catch JSpecify.** "Domain depends only on `java..`" fails on `@NullMarked`. Allow `org.jspecify.annotations..`: annotations only, no runtime behaviour.
- **Expect a false positive or two.** `ImmutableEnumChecker` flagged an enum holding stateless method references. Disable that check in the human-owned config, with a one-line comment saying why. Never with `@SuppressWarnings` in the code.
- **Pin the Error Prone version against your JDK.** A JDK upgrade is the usual thing that breaks it. 2.52.0 worked with JDK 25.

### ArchUnit

A JUnit test. Write the rules you keep repeating in code review.

```java
@AnalyzeClasses(packages = "com.acme")
class ArchitectureTest {
  @ArchTest static final ArchRule layers = layeredArchitecture().consideringAllDependencies()
      .layer("Web").definedBy("..web..")
      .layer("Service").definedBy("..service..")
      .layer("Persistence").definedBy("..persistence..")
      .whereLayer("Web").mayNotBeAccessedByAnyLayer()
      .whereLayer("Persistence").mayOnlyBeAccessedByLayers("Service");

  @ArchTest static final ArchRule noFieldInjection = GeneralCodingRules.NO_CLASSES_SHOULD_USE_FIELD_INJECTION;
  @ArchTest static final ArchRule noStdout = GeneralCodingRules.NO_CLASSES_SHOULD_ACCESS_STANDARD_STREAMS;

  // For anything that handles money or exact decimals.
  @ArchTest static final ArchRule noFloatingPoint = noClasses().should()
      .callMethod(BigDecimal.class, "doubleValue")
      .orShould().callMethod(BigDecimal.class, "floatValue")
      .orShould().callMethod(BigDecimal.class, "valueOf", double.class)
      .orShould().callConstructor(BigDecimal.class, double.class)
      .because("binary floating point loses the exactness the API promises");
}
```

It reads bytecode, so it works on Kotlin too. **Konsist** is the Kotlin-native alternative. It reads source, so it can check Kotlin-only things like `data class` usage and file naming.

The failure messages work as instructions. An agent reading `Class OrderController accesses OrderRepository, violating layer rule` knows exactly what to fix.

The gates overlap, and that's fine. In my first demo run, field injection was caught by Error Prone's `UnusedVariable` before ArchUnit got a look. Whichever gate fires first, the agent gets a precise message.

### PIT mutation testing

Coverage tells you a line ran. PIT changes the code (flips a `>` to `>=`, returns `null`, negates a condition) and checks that some test fails. Defect 11 had 100% line coverage, and its test killed none of the four mutants PIT made in it.

```kotlin
pitest {
    pitestVersion = "1.30.2"
    junit5PluginVersion = "1.2.3"
    targetClasses = setOf("com.acme.*")
    excludedClasses = setOf("com.acme.Application", "com.acme.config.*")
    threads = 4
    timestampedReports = false
    outputFormats = setOf("HTML", "XML")
    // 100% when added; the margin is for equivalent mutants, not for weak tests.
    mutationThreshold = 95
}
```

Set the threshold from a measured score, not a round number. Measure first, fix what PIT finds, then set the gate just below where you landed.

### Dependency rules

Gradle has no Enforcer, but the resolution strategy covers the important rules:

```kotlin
val bannedModules = mapOf(
    "junit:junit" to "JUnit 4; use JUnit Jupiter",
    "log4j:log4j" to "Log4j 1 is end of life; use SLF4J",
    "com.google.code.findbugs:jsr305" to "use JSpecify annotations",
)

configurations.configureEach {
    resolutionStrategy {
        failOnDynamicVersions()
        failOnChangingVersions()
        eachDependency {
            bannedModules["${requested.group}:${requested.name}"]?.let {
                throw GradleException("Banned dependency ${requested.group}:${requested.name}: $it")
            }
        }
    }
}
```

Don't ban `commons-logging` on Spring Framework 7: Spring depends on it directly now.

Maven:

```xml
<rules>
  <requireJavaVersion><version>[21,)</version></requireJavaVersion>
  <dependencyConvergence/>
  <requireUpperBoundDeps/>
  <banDynamicVersions/>
  <bannedDependencies><excludes>
    <exclude>junit:junit</exclude>
    <exclude>log4j:log4j</exclude>
  </excludes></bannedDependencies>
</rules>
```

Add `mvn dependency:analyze -DfailOnWarning=true` to catch dependencies that are declared but unused, or used but undeclared.

Run OWASP dependency-check (or let Renovate/Dependabot flag advisories). Agents sometimes invent package names that sound plausible, and attackers register those names ("slopsquatting").

### Database: Testcontainers, Flyway, Squawk

The demo project has no database, so this section isn't backed by the demo.

**Testcontainers.** Integration tests run against real Postgres, never H2:

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {
  @Container @ServiceConnection
  static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:17");
}
```

**Schema validation.** Set `spring.jpa.hibernate.ddl-auto=validate` in tests so an entity that doesn't match the migrated schema fails at startup. Flyway's default `validateOnMigrate` catches a migration that was edited after it was applied.

**Squawk** lints Postgres migrations for statements that lock tables or break running code: missing `CONCURRENTLY`, column renames, adding `NOT NULL` without a backfill:

```bash
git diff --name-only origin/main -- 'src/main/resources/db/migration/*.sql' | xargs -r squawk
```

### Kotlin

```kotlin
kotlin {
  compilerOptions {
    allWarningsAsErrors = true
    freeCompilerArgs.add("-Xjsr305=strict")
  }
  explicitApi() // libraries only
}

detekt {
  buildUponDefaultConfig = true
  baseline = file("detekt-baseline.xml")
}
```

- Run `./gradlew detektMain` (with type resolution), not plain `detekt`. Many of the useful rules need types.
- NullAway isn't needed, because Kotlin has null safety built in. Still put JSpecify annotations on your Java code: recent Kotlin compilers treat them as strict, so Kotlin calls into Java stay null-safe.
- Error Prone, Checkstyle and PMD don't run on Kotlin.

### Hygiene

- **Spotless**: `spotlessCheck` in the build, `spotlessApply` in the agent's instructions. It keeps PR diffs to real changes.
- **OpenRewrite**: a fixer, not a gate. `rewrite:dryRun` with `failOnDryRunResults=true` fails CI when the code doesn't match your recipes. It's also the best tool for framework and JDK upgrades.
- **Gitleaks**: first step in CI (`gitleaks/gitleaks-action@v2`, with `fetch-depth: 0` so it scans the whole history), and optionally pre-commit.

## Prove your gates, and keep proving them

The [demo harness](https://github.com/prule/math-api-outcome-based-backlog-openspec-quality-gates/tree/main/demo) is worth keeping after the blog post. Each defect is a short script: a heredoc that writes a file, plus a header naming the gate it should trip. The runner:

1. Exports the `baseline` tag and the current tree into two working copies, one per side, each with a throwaway git repo.
2. For each defect: `git reset --hard && git clean -fd` (build output stays, so `buildSrc` compiles once), runs the defect script, runs `spotlessApply` and then `build` (plus `pitest` if the defect asks for it).
3. Extracts the first line that names the gate from the log, and writes a Markdown table.
4. Pipes fake tool calls into the PreToolUse hook to check that it blocks edits to the gates and allows ordinary edits.

Run it when you upgrade Error Prone, change a rule, or move the build. A gate that quietly stops firing looks exactly like a gate that's passing.

One trap: my first version copied the project fresh for every defect. Recompiling `buildSrc` 22 times ran the Kotlin compile daemon out of memory, and those builds "failed" for the wrong reason, with empty logs. Check that a red result failed for the reason you expected.

## What not to bother with

- **Checkstyle**: Spotless handles formatting and Sonar handles the rest.
- **PMD**: the rules largely duplicate Sonar's. Keep it only if you need custom XPath rules.
- **SpotBugs**: optional. Overlaps with Error Prone and Sonar. Add **FindSecBugs** if you have no other security scanner.

## The point

Three AI reviews missed an NPE that a compiler plugin found in a second. Reviews are good at "is this the right change". Gates are good at "is this change correct in ways a machine can check". Make the gates fast, check them at the end of every turn and tell the human when they fail, lock them away from the agent, and test that they still fire.
