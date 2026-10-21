---
title: "One VM Instead of Seven Dev Containers"
date: 2026-09-19T15:35:00+10:00
publishDate: 2026-09-19
draft: false
layout: "post"
tags: ["devcontainer", "lima", "vm", "ai", "claude-code", "docker", "pnpm", "tooling"]
categories: ["coding"]
description: "All my dev work now runs in a single Lima VM, driven from the Claude Code desktop app over SSH. One sandbox, one toolchain, forty repos."
author: "Paul"
---

All my development now happens inside one Linux VM on my Mac. I drive it from the Claude Code desktop app over SSH, and it holds every one of my forty repos. The agent gets a proper sandbox, my Mac stays out of harm's way, and every project shares one toolchain.

It replaces the per-project setup from [Dev Containers for AI Agents]( {{< ref "post/coding/2026/devcontainers-for-ai-agents/" >}} ). The sandboxing idea is the same. There's just one box now instead of seven.

The idea is simple. I'm one person, on one laptop. The boundary I care about is between my Mac and the work the agent does. I don't need a wall between each project and every other project. So I put that one boundary in place once, around everything.

{{< notice type="info" title="The config" >}}
https://github.com/prule/devvm — one `dev.yaml`, three scripts. That's the whole environment.
{{< /notice >}}

Because everything lives in one place, a lot of things just work. There's one `~/.claude`, one `~/.config/gh`, one pnpm store and one Playwright cache. You log in once and stay logged in. Supabase is just `localhost`.

---

## How it fits together

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

You don't have to give up the GUI to work in a VM. The Claude Code desktop app treats SSH sessions as a first-class environment. Setting one up takes a minute:

1. Open the environment dropdown.
2. Pick *+ Add SSH connection*.
3. Give it a name, host, port and key.

The first time you connect, the app installs Claude Code on the VM for you.

From then on, the GUI stays on the Mac and everything runs in the VM. It's the same window, with the same permission prompts, file pane and diffs. If the agent runs `rm -rf node_modules` or a postinstall script misbehaves, it hits the VM's filesystem. Your `~/Documents` and your SSH keys are never in reach.

I think of it like this: the VM is the computer the agent uses, and the editor is a thin client attached to it.

Most features work over SSH: all four permission modes, the terminal panel, file editing, diffs and git, plugins, connectors, and MCP. Two things don't:

- `@mention` file autocomplete. You can still attach files.
- Shared session history. Each environment keeps its own.

## Lima

I use [Lima](https://lima-vm.io/) to run the VM. It's a good fit:

- It's free.
- The whole VM is defined in a config file.
- It supports `vmType: vz`, Apple's native hypervisor.
- It sets up Rosetta, so x86-only Docker images still run.

The best part is how nicely it plays with SSH. One command gives you a ready-made host entry:

```bash
limactl show-ssh dev --format=config >> ~/.ssh/config
```

That adds a `lima-dev` Host block, which is exactly what the desktop app's SSH dialog wants. Connecting the two takes about two minutes.

## The code lives in the VM

The VM has no mounts from the Mac. All the code lives inside it, and git moves changes in and out.

The reason is `node_modules`. It holds native binaries like esbuild, rollup and sharp, and those belong to whichever OS ran the install. If the code sat on a folder shared with the Mac, one stray `pnpm install` on the Mac would break every build in the VM. Your IDE also needs to read `node_modules` to resolve imports. So the IDE and the code need to live on the same side.

That means the IDE runs in the VM too. WebStorm's backend runs inside the VM, and JetBrains Gateway gives you the thin client on the Mac:

```bash
brew install --cask jetbrains-gateway
# Gateway -> SSH -> lima-dev -> pick a project
```

{{< notice type="tip" title="The result" >}}
Two thin clients run on the Mac: Claude Code desktop over SSH, and WebStorm over Gateway. Both point at the same box. One filesystem, one Node, one set of native binaries, and nothing to keep in sync.
{{< /notice >}}

The IDE backend does need a bit more room. I give the VM 6 CPUs and 14 GB of RAM. `dev.yaml` also sets up two things the backend expects from the OS:

- A few X and font libraries.
- A much higher `fs.inotify.max_user_watches`. Below about 500k, the IntelliJ platform quietly stops watching files and rescans directory trees instead.

If your IDE ever seems to reindex all the time, that sysctl is the one-line fix.

{{< notice type="warning" title="No snapshots" >}}
`vmType: vz` is fast, but `limactl snapshot` only works with QEMU, so there's no restore point. To recover, run `limactl delete dev && ./bootstrap.sh`. That only works if the YAML is complete. So treat any package in the VM that isn't in `dev.yaml` as a bug in `dev.yaml`, and skip the casual `sudo apt install`. I actually like this. It keeps the config honest.
{{< /notice >}}

## What's in the box

The VM comes with:

- `postgresql-client`, for raw-SQL migrations.
- fnm, for Node.
- Standalone pnpm.
- The OpenSpec CLI.
- SDKMAN, for Java (more on that below).
- Docker Engine, installed natively.

Docker runs straight on the VM's kernel. There's no Docker Desktop, no licence question and no VM inside a VM. My `supabase-local-platform` runs there too. It's in the same VM as everything else, so it's just `localhost`.

Every project shares the default pnpm store at `~/.local/share/pnpm/store`, so packages download once for the whole machine.

## One toolchain for everything

With one VM, every repo uses the same Node. Bumping Node updates all forty repos at once. That's the point, but it cuts both ways. If a repo doesn't build on the new version, it's telling you something about that repo, and it's worth fixing sooner rather than later.

fnm keeps one default Node version and is how I bump it.

pnpm makes this easy. From v10 onwards, it manages the `packageManager` field itself. So one standalone binary happily serves repos pinned to different pnpm versions.

### Java

Java is the one place where a single version isn't enough. Most of my CI runs on 25, so that's the default. But some modules pin `jvmToolchain(21)`, and Gradle treats that as a hard requirement. Without a real JDK 21 in the VM, Gradle either downloads one mid-build or fails.

SDKMAN handles every JDK, and apt installs none. That way there's exactly one answer to "which java is this?", just like fnm for Node. You list the versions in `dev.yaml`:

```yaml
JAVA_VERSIONS: "25.0.4-tem 21.0.12-tem"
JAVA_DEFAULT: "25.0.4-tem"
```

`JAVA_HOME` points at SDKMAN's `current` symlink. So `sdk default java <version>` changes the default for scripts too, from one place.

For a project that needs a different JDK, commit a `.sdkmanrc`:

```
java=21.0.12-tem
```

Now a `cd` into that directory switches the JDK. This works in **interactive shells only**, because it hooks `PROMPT_COMMAND`. Scripts and agent-run commands don't see it. For those, Gradle toolchains do the job. Gradle finds every SDKMAN-installed JDK, so `jvmToolchain(21)` resolves without downloading anything.

{{< notice type="warning" title="Three SDKMAN tips" >}}
**Install `zip`.** SDKMAN needs `zip`, not only `unzip`. Without it, the installer stops with "Please install zip on your system" and leaves no `~/.sdkman` at all.

**Set the default explicitly.** SDKMAN makes the *first* JDK you install the default. Always run `sdk default`, or interactive shells and scripts can quietly end up on different JDKs.

**Put the hardcoded `java` entry last in `PATH`.** `sdkman-init` adds its own entry, and `sdk use` changes *that* one. A hardcoded entry earlier in `PATH` wins every time, so `sdk env` reports one version while `java -version` reports another. At the end of `PATH`, the hardcoded entry only serves non-interactive shells, which have no `sdk` function anyway.
{{< /notice >}}

## One port tip

Lima forwards guest ports bound to `0.0.0.0` to the Mac automatically. Ports bound to `127.0.0.1` inside the VM stay hidden from the Mac by default, and that's what a default Vite config does. The fix is one line:

```ts
// vite.config.ts
export default defineConfig({
  server: { host: true },   // bind 0.0.0.0, not localhost
})
```

You can also declare `portForwards` in `dev.yaml`. I do that for the Supabase CLI, which hardcodes localhost. But binding to `0.0.0.0` always works.

## Wrapping up

The agent gets a real computer to run things on, and I can let it run things with confidence. My Mac stays clean. All of it is one YAML file and three shell scripts in a single repo.

The boundary that matters is between your machine and the work. You only need to draw it once.
