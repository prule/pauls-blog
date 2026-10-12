---
title: "One VM Instead of Seven Dev Containers"
date: 2026-09-19T15:35:00+10:00
publishDate: 2026-09-19
draft: true
layout: "post"
tags: ["devcontainer", "lima", "vm", "ai", "claude-code", "docker", "pnpm", "tooling"]
categories: ["coding"]
description: "Six weeks after standardising on dev containers, I deleted all of them and moved to a single Lima VM. The isolation was the right idea; one container per project just wasn't."
author: "Paul"
---

So it turns out I got the granularity wrong. The isolation part I still stand by.

Six weeks ago I wrote up [Dev Containers for AI Agents]( {{< ref "post/coding/2026/devcontainers-for-ai-agents/" >}} ): one container per project, the agent safely sandboxed, a reproducible toolchain. I still think all of that is right. But I've now deleted every one of those containers and replaced the whole lot with a single Linux VM, which I drive from the Claude Code desktop app over SSH.

The thing I kept bumping into is that I'm one person, working across forty repos, on one laptop. Isolating each project from every *other* project wasn't actually buying me anything. The boundary I cared about was between my host machine and the work the agent does. With one container per project I was maintaining forty boundaries when I only needed one.

{{< notice type="info" title="The config" >}}
https://github.com/prule/devvm — one `dev.yaml`, three scripts. That's the whole environment.
{{< /notice >}}

---

## The moment it clicked: ninety lines of workarounds

What finally tipped me over was looking at one project's `devcontainer.json` and seeing how much cruft had piled up in it:

| Workaround | Why it existed |
|---|---|
| `CLAUDE_CONFIG_DIR` | Claude Code splits state across `~/.claude/.credentials.json` and `~/.claude.json`. A volume covers one but not the other, so my logins vanished on every rebuild. |
| `GH_CONFIG_DIR` | The JetBrains backend repoints `XDG_CONFIG_HOME`, so `gh auth login` wrote to a throwaway dir instead of the mounted volume. |
| `PLAYWRIGHT_SKIP_BROWSER_GC` | Playwright's `.links` files hold absolute paths into a per-project `node_modules` volume. Sibling containers can't see those paths, so they decide the browsers are orphans and garbage-collect them. |
| `--add-host=host.docker.internal:host-gateway` | So I could reach the shared Supabase instance running on the host. |
| "Never mount a volume at `PNPM_HOME`" | A mount there shadows the baked-in standalone pnpm. |
| Shared `devcontainer-cache` / `-claude` / `-gh` volumes | Rebuilding "log in once per machine", which is a property a machine already has for free. |

When I read down that list, every single line was really the same thing: an attempt to make seven isolated environments behave like one environment.

At that point it's not a config problem any more. The design is trying to tell you what it actually wants to be.

{{< notice type="note" >}}
All six are gone now — not worked around, actually gone. There's one `~/.claude`, one `~/.config/gh`, one pnpm store, one Playwright cache, and Supabase is just `localhost`.
{{< /notice >}}

And then there was the disk. Docker Desktop's data directory had grown to 47 GB on a machine with 30 GB free. Meanwhile `~/projects` — the actual source code for all forty repos — is 15 GB. The containers had ballooned to three times the size of the thing they existed to build.

## The part I assumed wouldn't work

The obvious objection to a VM is that it means SSH, SSH means a terminal, and I wasn't about to give up the GUI.

As it happens, the Claude Code desktop app treats SSH sessions as a first-class environment. You go to the environment dropdown, pick *+ Add SSH connection*, give it a name, host, port and key, and it installs Claude Code on the remote for you on first connect.

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

So the GUI stays on the Mac and execution happens in the VM, in the same window, with the same permission prompts, the same file pane and diffs. When the agent runs `rm -rf node_modules` or a postinstall script fires, it hits the VM's filesystem, not `~/Documents` and not my host SSH keys.

That's the same mental model I landed on in the last post: the container is the computer the agent uses, and your editor is just a thin client attached to it. The only change is that I stopped running seven computers.

A fair bit works in SSH sessions: all four permission modes, the terminal panel, file editing, diffs and git, plugins, connectors, and MCP. A couple of things don't — `@mention` file autocomplete (attaching files still works), and session history is siloed per environment.

Losing `@mention` is the one thing that genuinely nags at me day to day. But I'll happily trade it for deleting 1,500 lines of config.

## Lima, and the one decision that matters

I went with Lima over UTM or Parallels because it's free, it's config-as-code, it supports `vmType: vz` for Apple's native hypervisor, and it sets up Rosetta binfmt so x86-only Docker images still run. Honestly, though, the deciding factor was pretty mundane:

```bash
limactl show-ssh dev --format=config >> ~/.ssh/config
```

That writes a real `lima-dev` Host block, which is exactly what the desktop app's SSH dialog wants to consume. The whole integration took about two minutes.

The decision that really matters, though, is this one:

```yaml
mounts: []
```

The code lives *in* the VM. Mounting `~/projects` from the host is tempting, and it's a trap. You get the filesystem performance penalty back, `node_modules` fills up with wrong-platform binaries, `.DS_Store` files appear everywhere, and you lose the isolation entirely, because the agent now has a path straight back to your home directory. A VM with a host mount is basically decoration.

Files move in and out by git, and that's it.

{{< notice type="warning" title="No snapshots" >}}
`vmType: vz` is fast, but `limactl snapshot` is QEMU-only, so there's no restore point. Recovery is `limactl delete dev && ./bootstrap.sh`, which only works if the YAML is complete. That means any package installed in the VM but missing from `dev.yaml` is a bug in `dev.yaml`, not a fact about the VM — so no casual `sudo apt install`. That discipline is the price of having no snapshots, and honestly I think it's a better property than snapshots would have been anyway.
{{< /notice >}}

## What happened to the Dockerfile

Most of it ported straight across: the `postgresql-client` for raw-SQL migrations, fnm, standalone pnpm, the OpenSpec CLI, and Temurin via apt. That last one incidentally killed off the `jdkDistro: tem` workaround, because apt doesn't choke on Microsoft's `21.0.12+1-ms` version strings the way the devcontainer Feature's regex did.

Docker Engine installs natively in the guest, so there's no Docker Desktop, no licence question, and no VM-inside-a-VM. Containers run on the guest kernel directly, which is where `supabase-local-platform` now lives — and since it's in the same VM as everything else, `host.docker.internal` is simply `localhost`.

The interesting part is what *didn't* port. The old `post-create.sh` had six steps, and three of them just don't exist any more:

```bash
# GONE — step 1: named volumes are created root-owned, and mounting one at
# ~/.cache/ms-playwright makes Docker create ~/.cache root-owned too
sudo chown vscode:vscode /cache/pnpm-store ~/.cache ~/.claude ~/.config/gh

# GONE — step 4: pnpm ignores PNPM_STORE_DIR and .npmrc for store-dir, and the
# JetBrains backend redirects XDG_CONFIG_HOME, so this had to be written twice
pnpm config set --global store-dir /cache/pnpm-store
XDG_CONFIG_HOME=/.jbdevcontainer/config pnpm config set --global store-dir /cache/pnpm-store
```

The default pnpm store at `~/.local/share/pnpm/store` is already shared by every project on the machine, which is all `/cache` was ever trying to be.

I marked each deletion with `GONE:` in the YAML at the spot where the ported step used to sit, so the omissions stay reviewable instead of quietly disappearing.

## "One toolchain" really does mean one

This is the part that isn't free. My repos were pinned all over the place:

- **Node**: `22`, `24.15.0`, `24.21.0`, `25.9.0`, `26.7.0`
- **pnpm**: `11.5.2` through `12.4.2`, six distinct pins

Each Dockerfile was effectively a toolchain lockfile for its own repo. With one VM, bumping Node bumps all forty at once. That's the synchronisation I was after, but it cuts both ways: a repo that won't build on Node 26 is now telling me something about that repo, rather than being a problem I can quietly defer forever.

pnpm turned out to be the pleasant surprise. From v10 onwards it self-manages the `packageManager` field, so a single standalone binary transparently serves repos pinned anywhere in that range. I install one and normalise the pins later.

fnm stays, but its role changed. It used to auto-switch per directory via `--use-on-cd`; now there's one default and fnm is just the mechanism I use to bump it.

Java is the one place the "one version" rule breaks down, and it's worth being honest about why. Most of my CI already runs on 25, so that's the VM default. But several `laptime-insights` modules pin `jvmToolchain(21)`, and a Gradle toolchain isn't a preference — it's a declared requirement. Without a real JDK 21 on the box, Gradle either downloads one mid-build or fails outright. So:

```yaml
JAVA_VERSIONS: "25 21"   # 25 is the default; 21 is here for the toolchains
JAVA_DEFAULT: "25"
```

Temurin co-installs cleanly from apt, so this costs a few hundred MB and nothing else. The general lesson is that "one version for everything" holds right up until a build tool treats a version as a hard input rather than a default.

## One gotcha worth knowing

Lima auto-forwards guest ports bound to `0.0.0.0`, but ports bound to `127.0.0.1` *inside* the guest are invisible from the host by default. That's exactly what a default Vite config does.

```ts
// vite.config.ts
export default defineConfig({
  server: { host: true },   // bind 0.0.0.0, not localhost
})
```

You can also declare explicit `portForwards` in `dev.yaml` instead — I do that for the Supabase CLI, which hardcodes localhost — but binding `0.0.0.0` is the fix that always works.

## So, was it worth it?

Last time I finished with the idea that the container is cheap, and the real win is the confidence to let the agent actually run things. That's still true, and nothing about the sandboxing argument has changed.

What changed is that I stopped paying for it forty times over. Seven `.devcontainer/` directories, seven copies of `docs/DevContainers.md`, a scaffolding skill to keep them all consistent, and 47 GB of Docker — all replaced by one YAML file and three shell scripts in a single repo.

The boundary that matters is host versus work. It turns out you only need to put it there once.
