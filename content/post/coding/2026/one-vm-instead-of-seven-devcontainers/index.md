---
title: "One VM Instead of Seven Dev Containers"
date: 2026-09-19T15:35:00+10:00
publishDate: 2026-09-19
draft: true
layout: "post"
tags: ["devcontainer", "lima", "vm", "ai", "claude-code", "docker", "pnpm", "tooling"]
categories: ["coding"]
description: "Six weeks after standardising on dev containers I deleted all of them for a single Lima VM — the isolation was right, the granularity was wrong."
author: "Paul"
---

I was wrong about the granularity. Not about the isolation.

Six weeks ago I wrote up [Dev Containers for AI Agents]( {{< ref "post/coding/2026/devcontainers-for-ai-agents/" >}} ) — one container per project, agent safely sandboxed, reproducible toolchain. All of that still holds. I've now deleted every one of them and replaced the lot with **a single Linux VM**, driven by the Claude Code desktop app over SSH.

The reason is simple: I'm one person working across forty repos on one laptop. Isolating each project from every *other* project bought me nothing. The boundary I actually cared about was between the host and the work. One container per project was paying for forty boundaries to get one.

{{< notice type="info" title="The config" >}}
https://github.com/prule/devvm — one `dev.yaml`, three scripts. That's the whole environment.
{{< /notice >}}

---

## The tell: ninety lines of workarounds

Here's what finally made the call obvious. One project's `devcontainer.json` had accumulated this:

| Workaround | Why it existed |
|---|---|
| `CLAUDE_CONFIG_DIR` | Claude Code splits state across `~/.claude/.credentials.json` and `~/.claude.json`. A volume covers one, not the other, so logins vanished on every rebuild. |
| `GH_CONFIG_DIR` | The JetBrains backend repoints `XDG_CONFIG_HOME`, so `gh auth login` wrote to a throwaway dir instead of the mounted volume. |
| `PLAYWRIGHT_SKIP_BROWSER_GC` | Playwright's `.links` files hold absolute paths into a per-project `node_modules` volume. Sibling containers can't see those paths, decide the browsers are orphans, and GC them. |
| `--add-host=host.docker.internal:host-gateway` | Reaching the shared Supabase instance on the host. |
| "Never mount a volume at `PNPM_HOME`" | A mount there shadows the baked-in standalone pnpm. |
| Shared `devcontainer-cache` / `-claude` / `-gh` volumes | Rebuilding "log in once per machine" — a property a machine already has. |

Every line is an attempt to make seven isolated environments behave like one environment.

That's not a config problem. That's the design telling you what it wants to be.

{{< notice type="note" >}}
All six are gone now. Not worked around — *gone*. There is one `~/.claude`, one `~/.config/gh`, one pnpm store, one Playwright cache, and Supabase is `localhost`.
{{< /notice >}}

And the disk. Docker Desktop's data directory had reached **47 GB** on a machine with 30 GB free. `~/projects` — the actual source code, forty repos — is 15 GB. The containers were three times the size of the thing they existed to build.

## The bit I assumed was impossible

The obvious objection: a VM means SSH, SSH means a terminal, and I'm not giving up the GUI.

Turns out the Claude Code desktop app **supports SSH sessions as a first-class environment**. Environment dropdown → *+ Add SSH connection* → name, host, port, key. It installs Claude Code on the remote itself on first connect.

{{< mermaid >}}
flowchart LR
    A[Claude Code desktop app] --> |SSH| B(Lima VM)

    subgraph sub1 [One VM - every project]
        C[Toolchain: Node/pnpm/JDK]
        D[~/projects - all 40 repos]
        E[Docker Engine + Supabase]
        F[~/.claude, ~/.config/gh]
    end

    B --> sub1
    A --> |forwarded ports| G[Host browser :5173]
    H[Host: Xcode, Finder] -.->|no mounts| sub1
{{< /mermaid >}}

GUI on the Mac. Execution in the VM. Same window, same permission prompts, same file pane and diffs. When the agent runs `rm -rf node_modules` or a postinstall script fires, it hits the VM's filesystem — not `~/Documents`, not my host SSH keys.

Which is exactly the mental model from the last post: *the container is the computer the agent uses, your editor is a thin client attached to it*. I just stopped running seven computers.

Supported in SSH sessions: all four permission modes, terminal panel, file editing, diffs and git, plugins, connectors, MCP. Not supported: `@mention` file autocomplete (attachment still works), and session history is siloed per environment.

Losing `@mention` is the one genuine daily annoyance. I'll take it to delete 1,500 lines of config.

## Lima, and one decision that matters

Lima over UTM or Parallels: free, config-as-code, `vmType: vz` for Apple's native hypervisor, Rosetta binfmt so x86-only Docker images still run. The deciding factor was mundane —

```bash
limactl show-ssh dev --format=config >> ~/.ssh/config
```

— which writes a real `lima-dev` Host block. Exactly what the desktop app's SSH dialog consumes. Two-minute integration.

The decision that actually matters is this one:

```yaml
mounts: []
```

**The code lives in the VM.** Mounting `~/projects` from the host is tempting and wrong. You get the filesystem performance penalty back, `node_modules` full of wrong-platform binaries, `.DS_Store` everywhere — and no isolation, because the agent now has a path straight back to your home directory. A VM with a host mount is decoration.

Files move by git. That's it.

{{< notice type="warning" title="No snapshots" >}}
`vmType: vz` is fast, but `limactl snapshot` is QEMU-only. There is no restore point. Recovery is `limactl delete dev && ./bootstrap.sh`, which only works if the YAML is complete — so **any package installed in the VM but not in `dev.yaml` is a bug in `dev.yaml`**, not a fact about the VM. No casual `sudo apt install`. That discipline is the price of no snapshots, and honestly it's a better property than snapshots would have been.
{{< /notice >}}

## What the Dockerfile became

Most of it ported straight across. The `postgresql-client` for raw-SQL migrations, fnm, standalone pnpm, the OpenSpec CLI, Temurin via apt (which incidentally kills the `jdkDistro: tem` workaround — apt doesn't choke on Microsoft's `21.0.12+1-ms` version strings the way the devcontainer Feature's regex did).

Docker Engine installs natively in the guest. No Docker Desktop, no licence question, no VM-inside-a-VM. Containers run on the guest kernel directly, which is where `supabase-local-platform` now lives — and because it's in the same VM as everything else, `host.docker.internal` is just `localhost`.

What *didn't* port is the interesting part. The old `post-create.sh` had six steps. Three of them don't exist any more:

```bash
# GONE — step 1: named volumes are created root-owned, and mounting one at
# ~/.cache/ms-playwright makes Docker create ~/.cache root-owned too
sudo chown vscode:vscode /cache/pnpm-store ~/.cache ~/.claude ~/.config/gh

# GONE — step 4: pnpm ignores PNPM_STORE_DIR and .npmrc for store-dir, and the
# JetBrains backend redirects XDG_CONFIG_HOME, so this had to be written twice
pnpm config set --global store-dir /cache/pnpm-store
XDG_CONFIG_HOME=/.jbdevcontainer/config pnpm config set --global store-dir /cache/pnpm-store
```

The default pnpm store at `~/.local/share/pnpm/store` is already shared by every project on the machine. That's all `/cache` ever wanted to be.

I marked each deletion `GONE:` in the YAML at the point where the ported step used to sit — so the omissions are reviewable rather than quietly forgotten.

## One toolchain means actually one

This is the part that isn't free. My repos were pinned all over the place:

- **Node**: `22`, `24.15.0`, `24.21.0`, `25.9.0`, `26.7.0`
- **pnpm**: `11.5.2` through `12.4.2`, six distinct pins

Each Dockerfile was effectively a toolchain lockfile for its repo. One VM means bumping Node bumps all forty at once. That's the synchronisation I wanted — it just cuts both ways, and a repo that won't build on Node 26 is now information about that repo rather than a thing I can defer forever.

pnpm was the pleasant surprise: v10+ self-manages the `packageManager` field, so one standalone binary transparently serves repos pinned anywhere in that range. Install one, normalise the pins later.

fnm stays, but its job changed. It used to auto-switch per directory via `--use-on-cd`. Now there's one default and fnm is just the mechanism for bumping it.

Java is the one place the rule doesn't hold, and it's worth being honest about why. Most of my CI is already on 25, so that's the VM default. But several `laptime-insights` modules pin `jvmToolchain(21)` — and a Gradle toolchain isn't a preference, it's a declared requirement. Without a real JDK 21 on the box, Gradle either downloads one mid-build or fails outright. So:

```yaml
JAVA_VERSIONS: "25 21"   # 25 is the default; 21 is here for the toolchains
JAVA_DEFAULT: "25"
```

Temurin co-installs cleanly from apt, so this costs a few hundred MB and nothing else. The lesson generalises: "one version for everything" holds right up until a build tool declares a version as a hard input rather than a default.

## Gotcha worth knowing

Lima auto-forwards guest ports bound to `0.0.0.0`. Ports bound to `127.0.0.1` **inside** the guest are invisible from the host by default — which is exactly what a default Vite config does.

```ts
// vite.config.ts
export default defineConfig({
  server: { host: true },   // bind 0.0.0.0, not localhost
})
```

You can declare explicit `portForwards` in `dev.yaml` instead (I do, for the Supabase CLI, which hardcodes localhost), but binding `0.0.0.0` is the fix that always works.

## Worth it?

Last time I ended with *the container is cheap; the confidence to let the agent actually run things is the win.* That's still the win. Nothing about the sandbox argument changed.

What changed is that I stopped paying for it forty times over. Seven `.devcontainer/` directories, seven `docs/DevContainers.md`, a scaffolding skill to keep them consistent, and 47 GB of Docker — replaced by one YAML file and three shell scripts, in one repo.

The boundary that matters is host versus work. Put it there once.
