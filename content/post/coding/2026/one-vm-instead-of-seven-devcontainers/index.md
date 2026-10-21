---
title: "One VM Instead of Seven Dev Containers"
date: 2026-09-19T15:35:00+10:00
publishDate: 2026-09-19
draft: true
layout: "post"
tags: ["devcontainer", "lima", "vm", "ai", "claude-code", "docker", "pnpm", "tooling"]
categories: ["coding"]
description: "Six weeks after standardising on dev containers, I deleted all of them and moved to a single Lima VM. The isolation was the right idea. One container per project was not."
author: "Paul"
---

It turns out I chose the wrong granularity. The isolation was still the right idea.

Six weeks ago I wrote [Dev Containers for AI Agents]( {{< ref "post/coding/2026/devcontainers-for-ai-agents/" >}} ): one container per project, the agent safely sandboxed, a reproducible toolchain. I still think all of that is right. But I deleted every one of those containers. I replaced them all with a single Linux VM, and I drive it from the Claude Code desktop app over SSH.

The problem I kept hitting is that I am one person, working across forty repos, on one laptop. Isolating each project from every *other* project gave me nothing. The boundary I cared about was between my host machine and the work the agent does. With one container per project, I maintained forty boundaries when I needed only one.

{{< notice type="info" title="The config" >}}
https://github.com/prule/devvm — one `dev.yaml`, three scripts. That is the whole environment.
{{< /notice >}}

---

## The moment it clicked: ninety lines of workarounds

The final push came from one project's `devcontainer.json`. I looked at how much cruft it contained:

| Workaround | Why it existed |
|---|---|
| `CLAUDE_CONFIG_DIR` | Claude Code splits state across `~/.claude/.credentials.json` and `~/.claude.json`. A volume covers one but not the other, so my logins vanished on every rebuild. |
| `GH_CONFIG_DIR` | The JetBrains backend repoints `XDG_CONFIG_HOME`, so `gh auth login` wrote to a throwaway dir instead of the mounted volume. |
| `PLAYWRIGHT_SKIP_BROWSER_GC` | Playwright's `.links` files hold absolute paths into a per-project `node_modules` volume. Sibling containers cannot see those paths, so they decide the browsers are orphans and garbage-collect them. |
| `--add-host=host.docker.internal:host-gateway` | So I could reach the shared Supabase instance on the host. |
| "Never mount a volume at `PNPM_HOME`" | A mount there shadows the baked-in standalone pnpm. |
| Shared `devcontainer-cache` / `-claude` / `-gh` volumes | They rebuilt "log in once per machine". A machine already has that property for free. |

Every line in that list was the same thing: an attempt to make seven isolated environments behave like one environment.

At that point it is not a config problem any more. The design is trying to tell you what it wants to be.

{{< notice type="note" >}}
All six are gone now. I did not work around them. They are gone. There is one `~/.claude`, one `~/.config/gh`, one pnpm store and one Playwright cache. Supabase is just `localhost`.
{{< /notice >}}

Then there was the disk. Docker Desktop's data directory grew to 47 GB on a machine with 30 GB free. Meanwhile `~/projects`, the source code for all forty repos, is 15 GB. The containers grew to three times the size of the code they existed to build.

## The part I assumed wouldn't work

The obvious objection to a VM is SSH. SSH means a terminal, and I would not give up the GUI.

But the Claude Code desktop app treats SSH sessions as a first-class environment:

1. Open the environment dropdown.
2. Pick *+ Add SSH connection*.
3. Give it a name, host, port and key.

On first connect, the app installs Claude Code on the remote for you.

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

The GUI stays on the Mac, and execution happens in the VM. It is the same window, with the same permission prompts, the same file pane and the same diffs. When the agent runs `rm -rf node_modules` or a postinstall script fires, it hits the VM's filesystem. It does not touch `~/Documents` or my host SSH keys.

That is the same mental model I reached in the last post. The container is the computer the agent uses, and your editor is a thin client attached to it. The only change is that I stopped running seven computers.

A lot works in SSH sessions: all four permission modes, the terminal panel, file editing, diffs and git, plugins, connectors, and MCP. Two things do not:

- `@mention` file autocomplete. Attaching files still works.
- Shared session history. Each environment keeps its own history.

Losing `@mention` is the one thing that annoys me day to day. But I will happily trade it for 1,500 fewer lines of config.

## Lima, and the one decision that matters

I chose Lima over UTM or Parallels for four reasons:

- It is free.
- It is config-as-code.
- It supports `vmType: vz` for Apple's native hypervisor.
- It sets up Rosetta binfmt, so x86-only Docker images still run.

Honestly, though, the deciding factor was mundane:

```bash
limactl show-ssh dev --format=config >> ~/.ssh/config
```

That command writes a real `lima-dev` Host block. The desktop app's SSH dialog wants exactly that block. The whole integration took about two minutes.

The decision that matters most is where the code lives. I changed my mind several times on this one, so I will show the whole loop and not only the answer.

**First position: purist.** `mounts: []`. The code lives inside the VM. Files move in and out by git, with no exceptions.

**Then I changed my mind.** The reason: **dev containers already bind-mount the workspace folder.** That is the default. So "never mount the host" did not protect a property I had for the last six weeks. I was inventing a new rule and calling it a principle. I also wanted WebStorm to run natively on the Mac, so I could watch files change as the agent wrote them. One writable mount of `~/projects/current` gives you that. It exposes only that one folder, and not `~/Documents`, `~/.ssh` or the keychain.

**Then I hit the thing that decides it.** `node_modules` would also be on the shared mount. So the native binaries (esbuild, rollup, sharp) belong to whichever OS ran the install last. My containers used per-project `node_modules` volumes specifically to stop that. The mount costs you that protection. That leaves you with a rule you must remember forever: *pnpm runs in the VM only, never on the host.* One `pnpm install` on the Mac breaks every build in the VM.

The obvious fix is to keep `node_modules` local to the VM with a bind mount. That is fast, with no mixing. But then WebStorm cannot resolve a single import. No config gives you both.

That split is the tell. The IDE needs `node_modules`, and `node_modules` can live safely in only one place. So **the IDE belongs on the same side of the boundary as the code.** I was trying to put a wall between two things that need to be in the same room.

So the answer is no mount. WebStorm's backend runs *in* the VM, and only the thin client runs on the Mac.

```bash
brew install --cask jetbrains-gateway
# Gateway -> SSH -> lima-dev -> pick a project
```

And of course, this line in my old `devcontainer.json` already did exactly that:

```json
"customizations": { "jetbrains": { "backend": "WebStorm" } }
```

I ran the IDE backend next to the code for six weeks, and I never thought of it as a design choice. The whole detour was me trying to solve a problem I already solved.

{{< notice type="tip" title="Where that leaves things" >}}
Two thin clients run on the Mac: Claude Code desktop over SSH, and WebStorm over Gateway. Both point at one box. There is one filesystem, one Node, one set of native binaries, and nothing to keep in sync. That is the devcontainer model I liked, with one container instead of seven.
{{< /notice >}}

It costs a bit of RAM. The backend is a real IDE process, so the VM went from 4 CPU / 12 GB to 6 / 14. `dev.yaml` must also provision two things that the backend wants from the OS:

- A few X and font libraries.
- A much higher `fs.inotify.max_user_watches`. Below about 500k, the IntelliJ platform quietly stops watching files and rescans directory trees instead.

That second one is the "why does my IDE keep reindexing" bug. The fix is a one-line sysctl.

{{< notice type="warning" title="No snapshots" >}}
`vmType: vz` is fast, but `limactl snapshot` is QEMU-only, so there is no restore point. Recovery is `limactl delete dev && ./bootstrap.sh`, which works only if the YAML is complete. So if a package is installed in the VM but missing from `dev.yaml`, that is a bug in `dev.yaml`, not a fact about the VM. No casual `sudo apt install`. That discipline is the price of no snapshots. Honestly, I think it is a better property than snapshots would have been anyway.
{{< /notice >}}

## What happened to the Dockerfile

Most of it ported directly: the `postgresql-client` for raw-SQL migrations, fnm, standalone pnpm, and the OpenSpec CLI. Java went somewhere else entirely (see below).

The `jdkDistro: tem` workaround died in the process, and the way it died is funny. That workaround existed because the devcontainer java Feature could not parse SDKMAN's version identifiers. `21.0.12+1-ms` broke its regex. Now I use SDKMAN directly, so I do not need to work around those identifiers. They are just what you type.

Docker Engine installs natively in the guest. So there is no Docker Desktop, no licence question, and no VM inside a VM. Containers run directly on the guest kernel, and `supabase-local-platform` now lives there. It is in the same VM as everything else, so `host.docker.internal` is simply `localhost`.

The interesting part is what did *not* port. The old `post-create.sh` had six steps, and three of them do not exist any more:

```bash
# GONE — step 1: named volumes are created root-owned, and mounting one at
# ~/.cache/ms-playwright makes Docker create ~/.cache root-owned too
sudo chown vscode:vscode /cache/pnpm-store ~/.cache ~/.claude ~/.config/gh

# GONE — step 4: pnpm ignores PNPM_STORE_DIR and .npmrc for store-dir, and the
# JetBrains backend redirects XDG_CONFIG_HOME, so this had to be written twice
pnpm config set --global store-dir /cache/pnpm-store
XDG_CONFIG_HOME=/.jbdevcontainer/config pnpm config set --global store-dir /cache/pnpm-store
```

Every project on the machine already shares the default pnpm store at `~/.local/share/pnpm/store`. That is all `/cache` ever tried to be.

I marked each deletion with `GONE:` in the YAML, at the spot where the ported step used to be. That way the omissions stay reviewable and do not quietly disappear.

## "One toolchain" really does mean one

This is the part that is not free. My repos had pins all over the place:

- **Node**: `22`, `24.15.0`, `24.21.0`, `25.9.0`, `26.7.0`
- **pnpm**: `11.5.2` through `12.4.2`, six distinct pins

Each Dockerfile was effectively a toolchain lockfile for its own repo. With one VM, a Node bump changes all forty repos at once. That is the synchronisation I wanted, but it cuts both ways. If a repo does not build on Node 26, that repo is now telling me something. It is no longer a problem I can quietly defer forever.

pnpm was the pleasant surprise. From v10 onwards, it self-manages the `packageManager` field. So a single standalone binary transparently serves repos pinned anywhere in that range. I install one binary and normalise the pins later.

fnm stays, but its role changed. It used to switch versions automatically per directory, via `--use-on-cd`. Now there is one default, and fnm is just the tool I use to bump it.

Java is the one place where the "one version" rule fails, and I want to be honest about why. Most of my CI already runs on 25, so 25 is the VM default. But several `laptime-insights` modules pin `jvmToolchain(21)`. A Gradle toolchain is not a preference. It is a declared requirement. Without a real JDK 21 on the box, Gradle either downloads one mid-build or fails outright.

At first I installed both from apt. That works, but it makes apt a second authority on Java versions, next to whatever tool I would use when I actually need to switch. So I gave the whole job to **SDKMAN** instead. apt installs no JDK at all:

```yaml
JAVA_VERSIONS: "25.0.4-tem 21.0.12-tem"
JAVA_DEFAULT: "25.0.4-tem"
```

That mirrors fnm, which owns every Node version with no apt Node beside it. One authority per language means "which java is this?" has exactly one answer. `JAVA_HOME` points at SDKMAN's `current` symlink. So `sdk default java <version>` moves the default for scripts too, from one place.

{{< notice type="warning" title="Three things that appeared only on the real machine" >}}
I wrote this config. It parsed, and the scripts passed `bash -n`. All three of these problems still bit me the moment it ran:

**SDKMAN needs `zip`.** My package list had `unzip` but not `zip`. The installer stops with "Please install zip on your system" and leaves no `~/.sdkman` at all.

**SDKMAN makes the *first* JDK you install the default.** I installed 21 first. So interactive shells got 21 while scripts got 25. That split-brain looks like nothing at all until a build picks the wrong JDK. `sdk default` must be explicit.

**The `java` entry in `PATH` must come last.** This one took a while to find. `sdk env` cheerfully reported "Using java version 21.0.12-tem in this shell", and `java -version` still said 25. `sdkman-init` prepends its own entry, and `sdk use` rewrites *that* entry. So my hardcoded entry earlier in `PATH` quietly won every time. At the end of `PATH`, my entry serves only non-interactive shells, and those have no `sdk` function anyway.
{{< /notice >}}

In each project, you commit a `.sdkmanrc`:

```
java=21.0.12-tem
```

…and a `cd` into the directory switches the JDK. One caveat: this works in **interactive shells only**, because it hooks `PROMPT_COMMAND`. Scripts and agent-run commands never see it. For those, Gradle toolchains are the real mechanism. Gradle finds every SDKMAN-installed JDK in the candidates directory. So a declared `jvmToolchain(21)` resolves without a download mid-build.

The general lesson still holds. "One version for everything" works until a build tool treats a version as a hard input and not as a default. The narrower lesson: a version manager and a hardcoded `PATH` are two authorities on the same question, and the hardcoded one wins silently.

## One gotcha worth knowing

Lima automatically forwards guest ports bound to `0.0.0.0`. But by default, the host cannot see ports bound to `127.0.0.1` *inside* the guest. That is exactly what a default Vite config does.

```ts
// vite.config.ts
export default defineConfig({
  server: { host: true },   // bind 0.0.0.0, not localhost
})
```

You can also declare explicit `portForwards` in `dev.yaml` instead. I do that for the Supabase CLI, which hardcodes localhost. But binding to `0.0.0.0` is the fix that always works.

## So, was it worth it?

Last time, I ended with this idea: the container is cheap, and the real win is the confidence to let the agent actually run things. That is still true. Nothing about the sandboxing argument changed.

What changed is that I stopped paying for it forty times over. All of this is gone:

- Seven `.devcontainer/` directories.
- Seven copies of `docs/DevContainers.md`.
- A scaffolding skill to keep them all consistent.
- 47 GB of Docker.

One YAML file and three shell scripts in a single repo replaced all of it.

The boundary that matters is host versus work. It turns out you need to put it there only once.
