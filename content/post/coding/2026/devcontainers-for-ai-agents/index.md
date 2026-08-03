---
title: "Dev Containers for AI Agents"
date: 2026-07-26T14:34:47+10:00
publishDate: 2026-07-26
draft: true
layout: "post"
tags: ["devcontainer", "docker", "ai", "claude-code", "pnpm", "gradle", "tooling"]
categories: ["coding"]
description: "Why every project I run an AI agent in now lives in a Dev Container — the sandbox, the reproducibility, and the standard I settled on."
author: "Paul"
---

I can't believe I haven't been using [devcontainers](https://containers.dev) before now - A simple configuration allows me to do development from Intellij/WebStorm/VSCode inside a container - this container is configured with the right versions of the right tools, and because it's containerised I can safely put claude code in auto mode removing the need to check and approve every command. The consistency this containerisation provides makes it even more beneficial on teams where multiple people need consistent environments. 

## Why it matters for agents

**Blast radius.** An agent runs shell commands. In a container, "run the tests" or "install this" or a hallucinated `rm` hits a throwaway Linux box, not your laptop. Reset is `docker rm -f` plus a rebuild, not a reinstall of your machine.

**It can close its own loop.** The interesting agent workflows aren't "generate a file". They're propose → implement → **run** → observe → fix. That needs a real environment: a dev server, a database, a browser for end-to-end tests. The container *is* that environment, always present, identical every time.

**Reproducible, so failures are real.** The container matches CI and the deploy target. When the agent says "tests pass", they pass on the same Node, same pnpm, same JDK that CI will use — not a version drifted three minors ahead on my host. "Works on my machine" stops being a category of bug, for me and for the agent.

**Parallelism.** Static-named, self-contained boxes are easy to run several of. Point agents at different projects (or branches) without their toolchains colliding.

**The tools travel with the repo.** The agent's own CLI, the spec workflow, `gh` for PRs — all provisioned by the container, not assumed to exist. New machine, or a teammate, or a fresh agent session: same environment, zero setup beyond Docker and an editor. The tools are versioned with the git repository.

{{< notice type="tip" >}}
The mental model: the container is the computer the agent uses. Your editor (VS Code or a JetBrains IDE) is a thin client attached to it. You and the agent share the same box.
{{< /notice >}}

## How it works

Three files in `.devcontainer/` describe the box:

- **`Dockerfile`** — the image: base OS plus the toolchain you build yourself (in my case Node via `fnm`, standalone `pnpm`, the OpenSpec CLI).
- **`devcontainer.json`** — the wiring: which prebuilt [Features](https://containers.dev/features) to layer on (Claude Code, GitHub CLI, docker-in-docker…), which ports to forward, which volumes to mount, the container name, editor settings.
- **`post-create.sh`** — one-time provisioning after the image is built: install dependencies, fetch the Playwright browser, seed a `.env`.

The editor (or the `devcontainer` CLI) reads these, builds the image, starts the container, and drops you — and the agent's terminal — inside it.

{{< mermaid >}}
flowchart LR
    A[You] --> |thin client| B(Dev Container)

    subgraph sub1 [Dev Container - one per project]
        C[Toolchain: Node/pnpm/JDK]
        D[App + dev server + DB]
        E[Agent CLI, gh, OpenSpec]
    end

    B --> sub1
    B --> |mount| F[("Shared caches /cache")]
    B --> |matches| G[CI / Deploy target]
{{< /mermaid >}}

## The standard I settled on

I rolled this out across five projects — a couple of React/Vite PWAs, a Kotlin/Ktor + React monorepo, and a Supabase app — then wrote it down as a [playbook page](https://prule.github.io/project-setup/DevContainers/) and an installable skill so future projects get it for free. The shape:

- **Node via [`fnm`]( {{< ref "post/coding/2026/switching-to-fnm" >}} )**, driven by `.node-version`. Not corepack.
- **[`pnpm`]( {{< ref "post/coding/2026/why-switch-to-pnpm" >}} ) installed standalone**, pinned to the repo's `packageManager`.
- **[OpenSpec]( {{< ref "post/coding/2026/open-spec" >}} ) CLI + Claude Code + GitHub CLI** — the agent's workflow tools, provisioned in the image.
- **JDK from the base image** for the Kotlin/Gradle project; **Gradle** via the wrapper.
- **Playwright + Chromium** for end-to-end tests the agent can actually run.

### Shared caches: download once per machine

A single Docker volume - `devcontainer-cache` mounted at `/cache` - is shared by *every* project's container. It holds the pnpm store (`/cache/pnpm-store`) and the Gradle cache (`/cache/gradle`, via `GRADLE_USER_HOME`). A package one project downloads is already there for the next (provided they use the same versions).

```jsonc
// devcontainer.json
"mounts": [
  "source=devcontainer-cache,target=/cache,type=volume"
],
"containerEnv": {
  "GRADLE_USER_HOME": "/cache/gradle"
}
```

```bash
# post-create.sh — pnpm only honours this via its global config
pnpm config set --global store-dir /cache/pnpm-store
```

Heavy, platform-specific things stay **per-project** and disposable instead — `node_modules`, the Playwright browsers, the nested-Docker images for the Supabase stack. Never on the host bind mount, so a Linux container never inherits a macOS-built native binary.

## Lessons from the rollout

A few that cost real time, in case they save you some:

- **The OpenSpec CLI is `@fission-ai/openspec`.** The bare `openspec` on npm is an empty placeholder with no binary. Install the wrong one and every spec command silently no-ops.
- **A `SQLite disk I/O error` from pnpm meant Docker Desktop's disk was full**, not a config bug. Four container images plus their volumes fill 60GB faster than you'd think. `docker system df`, then prune build cache and unused images.
- **The `java` base image ships an expired yarn apt key** that breaks every `apt-get update` — and therefore Playwright's `--with-deps`. One `rm` of the offending source file fixes it.
- **Verify in an interactive shell.** A non-login `bash -c` doesn't source `.bashrc`, so `fnm` never activates and everything looks "not installed". `bash -ic` is how the agent's terminal actually behaves.

## Worth it?

Once the agent could run and observe its own changes inside a box I didn't have to babysit, the work shifted from me driving every command to me reviewing outcomes. The reproducibility and the shared caches are a nice tax cut on top. The container is cheap; the confidence to let the agent actually *run* things is the win.

> See https://github.com/prule/intentionhorizon for one of my sample projects that uses devcontainers and openspec
> * `.devcontainers/` - contains the devcontainers setup - open devcontainers.json in Intellij and right click in the top left hand side to open 
> * `openspec/` - contains the config.yaml, specs, and an archive of the changes that have happened
