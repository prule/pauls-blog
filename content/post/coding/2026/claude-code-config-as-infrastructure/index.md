---
title: "Claude Code Config as Infrastructure: Rebuilt on Every Boot"
date: 2026-11-28T16:20:00+11:00
publishDate: 2026-11-28
draft: false
layout: "post"
tags: ["claude-code", "ai", "devvm", "lima", "skills", "plugins", "claude-md"]
categories: ["coding"]
description: "How my dev VM writes Claude Code's instructions, skills and plugins from one config file on every boot, so a rebuilt VM gets the same agent setup and updates arrive without me doing anything."
author: "Paul"
---

My [dev VM](https://github.com/prule/devvm) has one rule: if something is installed in the VM and not in `dev.yaml`, that's a bug in `dev.yaml`. There are no snapshots. Recovering means deleting the VM and building it again from that file, which only works if the file is complete.

Claude Code's own setup took a few rounds to meet that rule. Skills and plugins were in `dev.yaml` early on, but only installed once, so the VM kept whatever version it had first installed. The `CLAUDE.md` files came later still.

Now all of it comes from `dev.yaml`, and every boot puts it back and brings it up to date:

| What | Where it ends up | How |
|---|---|---|
| User-wide instructions | `~/.claude/CLAUDE.md` | rewritten from `dev.yaml` |
| Standards for every project | `~/projects/CLAUDE.md` | rewritten from `dev.yaml` |
| Skills from GitHub repos | `~/.claude/skills/` | installed if missing, then updated |
| Skills written in `dev.yaml` | `~/.claude/skills/` | rewritten from `dev.yaml` |
| Plugins | `~/.claude/settings.json` + installed | enabled, installed if missing, then updated |

Agent instructions are configuration. They get the same treatment as the Node version.

## Instructions, rewritten on every boot

Claude Code reads `~/.claude/CLAUDE.md` in every session. It also reads `CLAUDE.md` files in parent directories, so `~/projects/CLAUDE.md` applies to every project under `~/projects`. Both are written from `dev.yaml`.

The user-wide one holds rules for how Claude works anywhere. For example:

> Never bypass a git hook with `--no-verify` (or `-n`) unless the user asks for it. If a hook blocks a commit, such as the betterleaks secret scan, fix the cause; if it looks like a false positive, tell the user and ask before adding a `betterleaks:allow` comment or a `.betterleaksignore` entry.

The projects one holds the engineering standards: run the tests and `sonar-check` before reporting done, formatting inside the test command, pinned dependencies, and the per-language rules.

Each file starts with a comment saying where it comes from:

```markdown
<!-- Generated from devvm/dev.yaml on every boot. Edit it there; changes made here are lost on restart. -->
```

That's for me and for Claude. Without it, an edit made in the VM would look like it worked, then quietly vanish on the next restart.

There's one small trick in how they're written. Lima's `data` provisioning mode writes files as root. Writing straight into `~/.claude` on a fresh VM could create that directory owned by root, before anything else exists there. So the file is staged in `/usr/local/share/devvm/` and a user-mode step copies it into place:

```bash
mkdir -p "$HOME/.claude"
install -m 644 /usr/local/share/devvm/CLAUDE.md "$HOME/.claude/CLAUDE.md"
```

## Skills: install, then update

Skills come from GitHub repos, installed with the `skills` CLI (its version pinned in `dev.yaml` and bumped by Renovate):

```bash
# "<github owner/repo> <skill name>", one per line. Every skill in the
# repo is installed; the name is only the "already there?" sentinel.
skills='
danyuchn/asd-ste100-skill asd-ste100
mattpocock/skills setup-matt-pocock-skills
prule/project-setup vssw-audit-project-compliance
'
```

The first version of this step only installed skills that were missing. That got them onto a new VM, but a change pushed to a skill repo, like my own `prule/project-setup`, never reached a VM that already had it. So now every boot also runs an update:

```bash
npx --yes "skills@${PARAM_SKILLS_CLI_VERSION}" update -g -y
```

`skills add` records each skill's source and a hash of its folder. `update` compares those hashes with GitHub and re-copies only the skills that changed. It also overwrites local edits to a skill, which is exactly what I want here: the repo is the source of truth.

Some skills are specific to this VM, so they're written in `dev.yaml` itself and copied into `~/.claude/skills` the same way as the `CLAUDE.md` files. The `apply-project-standards` skill is one: it brings a repo in line with the standards in `~/projects/CLAUDE.md`. Keeping the rules and the skill that applies them in the same file keeps them in step.

## Plugins: enabling isn't installing

Plugins took the most attempts to get right, and each fix is in the commit history.

**Attempt one: enable them in `settings.json`.** The boot merges each plugin's marketplace and an `enabledPlugins` entry into `~/.claude/settings.json` with `jq`, so my other settings survive:

```bash
jq --arg m "$market" --arg r "$repo" --arg p "$plugin@$market" '
  .extraKnownMarketplaces[$m] = {source: {source: "github", repo: $r}, autoUpdate: true}
  | .enabledPlugins[$p] = true
' "$settings" > "$tmp" && mv "$tmp" "$settings"
```

**Problem: they never updated.** Third-party marketplaces don't auto-update by default, so the VM kept the first version of a plugin it installed. Setting `autoUpdate: true` on the marketplace entry fixed that for new sessions, but `autoUpdate` only acts when a session starts, and the existing VM sat on version 0.6.0 of my sdlc-pipeline plugin while 0.8.0 was out. So every boot now also runs `claude plugin marketplace update` and `claude plugin update`.

**Problem: a new plugin never arrived.** I added ponytail to the list, and it never showed up. Enabling a plugin in `settings.json` doesn't install it, and `plugin update` fails on a plugin that was never installed. So the step now checks Claude Code's own records first, and adds a missing marketplace and installs a missing plugin before updating the rest:

```bash
if ! jq -e --arg m "$market" 'has($m)' "$known" > /dev/null 2>&1; then
  timeout 180s "$claude" plugin marketplace add "$repo" < /dev/null
fi
if ! jq -e --arg p "$plugin@$market" '.plugins | has($p)' "$installed" > /dev/null 2>&1; then
  timeout 180s "$claude" plugin install "$plugin@$market" --scope user < /dev/null
fi
```

Adding a plugin is now one line in `dev.yaml`'s list, and the next boot does the rest.

## Which `claude`?

The plugin commands need a `claude` CLI, and that turned out to be its own small problem. The desktop app installs its copy under `~/.claude/remote/ccd-cli/<version>`, which isn't on `PATH` and changes name with every update. So a shell in the VM had no `claude` at all.

The VM now runs Claude's native installer into `~/.local/bin`, which updates itself and so isn't pinned. The plugin step uses whichever CLI it can find, falling back to the newest desktop-app copy:

```bash
claude="$(command -v claude || command -v "$HOME/.local/bin/claude" || ls -d "$HOME"/.claude/remote/ccd-cli/* 2>/dev/null | sort -V | tail -1 || true)"
[ -n "$claude" ] || exit 0
```

If there's no CLI yet, the step stops quietly. The plugins are still enabled in `settings.json`, and the next boot installs them.

## Never fail the boot

Every one of these steps is best-effort. A GitHub hiccup, no network, or no GitHub key yet on a fresh VM must not stop the VM from booting. So failures print a line starting with `!` and carry on:

```bash
|| echo "! plugin $plugin not updated — run '$claude plugin update $plugin@$market' later"
```

That's easy to miss in a boot log, so the script that applies `dev.yaml` to the VM finishes by pulling out this boot's `!` lines and showing them. A skipped skill or plugin is in front of me, with the command to fix it.

## What it adds up to

Restarting the VM gives me the current instructions, the latest skills, and the latest plugins, all from one file in git. A rebuild from scratch gets the same. Changing how Claude works across every project means editing `dev.yaml` and restarting, and the change has a commit message saying why.
