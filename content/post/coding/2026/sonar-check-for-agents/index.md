---
title: "sonar-check: Making SonarQube Part of an Agent's Definition of Done"
date: 2026-11-26T15:40:00+11:00
publishDate: 2026-11-26
draft: false
layout: "post"
tags: ["sonarqube", "static-analysis", "claude-code", "ai", "devvm", "quality-gates", "docker"]
categories: ["coding"]
description: "How my dev VM runs SonarQube locally and wraps the scanner in a command that fails on a red quality gate and prints the issues as file:line, so coding agents fix them before they say done."
author: "Paul"
---

In my [quality gates post](/pauls-blog/post/coding/2026/ai-code-quality-gates-java-kotlin/) I said SonarQube alone won't stop an agent shipping bad code, because Sonar reports and agents don't read dashboards. That's still true. But it doesn't mean Sonar is useless to an agent. It means the agent needs Sonar as a command that fails, with output it can act on.

So my [dev VM](https://github.com/prule/devvm) runs SonarQube locally and has a small wrapper called `sonar-check`. Run it in any repo, and it scans, waits for the quality gate, and exits non-zero if the gate fails. When it fails, it prints the problems in this shape (an illustrative example):

```text
=== sonar-check: FAILED for my-repo (http://localhost:9000/dashboard?id=my-repo) ===
--- failed quality gate conditions ---
new_violations: 3 (fails when GT 0)
--- open issues in new code (file:line rule message) ---
src/orders/OrderService.java:42  java:S2259  A "NullPointerException" could be thrown; "order" is nullable here.
```

That's something an agent can work with. And every project's `CLAUDE.md` standards say:

> Before reporting work as done, run the project's tests and `sonar-check`, and fix what fails.

## The server

SonarQube Community Build runs in Docker inside the VM, published on `127.0.0.1:9000` only:

```bash
docker run -d --name sonarqube --restart unless-stopped \
  -p 127.0.0.1:9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  "sonarqube:${PARAM_SONARQUBE_VERSION}"
```

It needs `vm.max_map_count` raised to 524288 for its embedded Elasticsearch, which the VM sets through `/etc/sysctl.d`. `unless-stopped` brings it back at boot, but leaves it down if I've stopped it on purpose.

The scanner CLI is installed from SonarSource's zip, which comes with its own JRE. That's handy: it ignores whichever Java SDKMAN currently has selected.

Both versions are pinned in `dev.yaml`, and Renovate bumps them.

## The wrapper

The whole of `sonar-check` is about 40 lines of bash. The interesting parts:

**It works out the project on its own.** A `sonar.projectKey` in `sonar-project.properties` wins; otherwise the repo's directory name is the key. A repo needs no config to start, and the first scan creates the project in SonarQube.

```bash
root="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
key="$(sed -n 's/^sonar\.projectKey=//p' "$root/sonar-project.properties" 2>/dev/null || true)"
key="${key:-$(basename "$root")}"
```

**It waits for the gate.** `-Dsonar.qualitygate.wait=true` makes the scanner block until SonarQube has processed the analysis, and exit non-zero if the quality gate failed. Without it, the scanner exits as soon as it has uploaded, and you'd have to poll.

```bash
(cd "$root" && SONAR_TOKEN="$token" SONAR_HOST_URL="$host" \
  sonar-scanner -Dsonar.projectKey="$key" -Dsonar.qualitygate.wait=true "$@") || status=$?
```

**When it fails, it fetches the reasons.** A failed gate on its own says "no" without saying why. So the wrapper calls two SonarQube APIs and formats the results with `jq`: the failed gate conditions, and the open issues in new code as `file:line rule message`.

```bash
curl -fsSu "$token:" "$host/api/issues/search?components=$key&resolved=false&inNewCodePeriod=true&ps=100" \
  | jq -r '.issues[] | "\(.component | sub("^[^:]*:"; "")):\(.line // "-")  \(.rule)  \(.message)"'
```

`file:line` is the format every tool and editor understands, agents included. The agent reads the line, opens the file, fixes it, and runs `sonar-check` again.

**It fails clearly when it can't run.** No token, or the server isn't up, and it exits with code 2 and a message saying what to do (`docker start sonarqube`, say), rather than a scanner stack trace.

Extra arguments go straight through to the scanner, so `sonar-check -Dsonar.exclusions=dist/**` works.

## The token

The one manual step. After a fresh VM, I log into SonarQube once, change the admin password and generate a **user token**, which goes into `~/.config/sonar/token` with only me able to read it.

It has to be a user token, not a project analysis token. A user token carries the admin's permission to create projects, which is what lets the first scan of a new repo create its project. A project token only works for a project that already exists.

## Using it as a gate

There are two places it plugs in.

**The project standards.** The line in `~/projects/CLAUDE.md` above applies to every session in every project. "Done" means the tests passed and the quality gate passed, not just that the code was written.

**My sdlc-pipeline.** Every agent in the pipeline (implementer, QA, senior dev) runs the repo's Verify command and has to make it pass. So `sonar-check` goes on the end of it:

```markdown
- **Verify:** `pnpm test && sonar-check`.
  Every agent runs this to prove its work and pastes the output. A failed SonarQube quality gate
  is a failed Verify: fix the listed issues, don't suppress them with `// NOSONAR`.
```

It goes last because tests are faster and should fail first. A scan takes from 30 seconds to a few minutes.

The "don't suppress them" line matters. The easiest way for an agent to make a finding go away is `// NOSONAR`. Saying in advance that a suppression isn't a fix closes that door.

{{< notice type="tip" >}}
Run `sonar-check` once on the main branch when you start using it on a repo. The quality gate only judges *new code*, which by default means anything changed since the first scan. The first scan sets that baseline and passes whatever it finds. If it happens in the middle of an agent's run, that run's changes get a free pass.
{{< /notice >}}

## What it costs

**Memory.** SonarQube takes about 2–3 GB while it's running. I'd lowered the VM from 14 GiB to 10 GiB to leave more room for macOS and Xcode, and SonarQube ran the VM out of memory, so it went back to 14. When I'm not using it, `docker stop sonarqube` keeps it down, even across reboots.

**One analysis per repo.** The Community Build has no branch or pull-request analysis. Each scan replaces the last, whatever branch it ran on. Fine for one person working one branch at a time.

**Version bumps start fresh.** The container uses SonarQube's embedded H2 database, which can't be upgraded in place. A version bump means a fresh database and redoing the token setup. That's a few minutes, and I don't keep any history in it worth saving.
