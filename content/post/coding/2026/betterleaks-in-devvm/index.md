---
title: "Secret Scanning Every Commit in My Dev VM with Betterleaks"
date: 2026-11-22T13:27:45+11:00
publishDate: 2026-11-22
draft: true
layout: "post"
tags: ["betterleaks", "gitleaks", "security", "git-hooks", "husky", "lima", "claude-code", "ai"]
categories: ["coding"]
description: "Why I wanted every commit in my dev VM scanned for secrets, the three places I had to put a git hook to make that true, and what changed when I moved from Gitleaks to Betterleaks."
author: "Paul"
---

A while ago I moved all my projects into [one Linux VM](https://github.com/prule/devvm), and these days most of the commits in it aren't typed by me. Claude Code writes the code, runs the tests and commits. That's mostly great, but I wanted to be sure no secrets ever got pushed. So I set out to make sure every commit in the VM got scanned for secrets before it was made. Every repo, including ones I'd cloned months ago and forgotten about.

It turned out to be a bit more fiddly than I expected.

## Starting simple

My first thought was the obvious one: a `pre-commit` hook that runs a secret scanner on the staged changes. At the time that was [Gitleaks](https://github.com/gitleaks/gitleaks). These days it's [Betterleaks](https://github.com/betterleaks/betterleaks), its successor from the same people. More on the switch later.

The command itself is easy:

```bash
betterleaks git --pre-commit --staged
```

The hard part was getting that hook into every repo. A git hook lives in `.git/hooks/`, which isn't committed, so every clone starts without one. I didn't want to set it up repo by repo. I'd forget, and the agents would never think to.

## Three ways in

I ended up needing three separate ways in, because each one misses something the others catch.

**New repos** were the easy part. Git has a template directory: whatever is in it gets copied into `.git/` when you `git clone` or `git init`. So the VM sets that up system-wide:

```bash
git config --system init.templateDir /usr/local/share/devvm/git-template
```

I used `--system` on purpose. It writes to `/etc/gitconfig` and leaves my own `~/.gitconfig` alone.

**Existing repos** don't benefit from the template, because it only applies at clone time. So every time the VM boots, a script walks `~/projects` and adds the hook to any repo that doesn't have one:

```bash
find "$HOME/projects" -maxdepth 4 -name .git -type d -prune 2>/dev/null | while read -r gitdir; do
  repo="$(dirname "$gitdir")"
  # Husky (or another hooks dir) is set: .git/hooks is ignored there.
  [ -n "$(git -C "$repo" config --local core.hooksPath 2>/dev/null)" ] && continue
  hook="$gitdir/hooks/pre-commit"
  if [ ! -e "$hook" ]; then
    mkdir -p "$gitdir/hooks"
    install -m 755 /usr/local/share/devvm/git-template/hooks/pre-commit "$hook"
  elif ! grep -q 'devvm-secret-scan' "$hook"; then
    echo "! ${repo#$HOME/}: has its own pre-commit hook, so commits there are NOT scanned for secrets"
  fi
done
```

I had to decide what to do when a repo already had its own `pre-commit` hook (lefthook, the pre-commit framework, something hand-written). Overwriting it felt worse than the gap it would close, so the script leaves those repos alone and names them in the boot output. That way I know which repos aren't covered.

`-type d` is there because of worktrees. A worktree's `.git` is a file, not a directory, and it shares the main repo's hooks anyway.

**Husky repos** needed their own fix. Husky sets `core.hooksPath`, and once that's set git ignores `.git/hooks` completely.

Luckily Husky v9 has a hook of its own: before it runs any hook, it sources `~/.config/husky/init.sh`, with the hook's name in `$n`. So the boot adds one line to that file, `. /usr/local/share/devvm/husky-init.sh`, and that file says:

```sh
# Added by devvm: secret scan before every commit (devvm README, Secret scanning).
if [ "${n:-}" = pre-commit ]; then
  /usr/local/bin/devvm-secret-scan || exit 1
fi
```

It's a sourcing line rather than a copy, so anything else I've put in `init.sh` survives, and a change to `husky-init.sh` takes effect straight away.

{{< mermaid >}}
flowchart LR
    A[git commit] --> B{core.hooksPath set?}
    B -- no --> C[.git/hooks/pre-commit<br/>from template or boot step]
    B -- Husky --> D[~/.config/husky/init.sh]
    C --> E[devvm-secret-scan]
    D --> E
    E --> F[betterleaks git --pre-commit --staged]
    F -- clean --> G[commit]
    F -- finding --> H[blocked + how to fix]
{{< /mermaid >}}

## One script behind all of them

If each hook ran the scanner directly, changing a flag would mean updating a copy of the hook in every repo. So every hook is a tiny shim:

```sh
#!/bin/sh
# Added by devvm: secret scan before every commit (devvm README, Secret scanning).
exec /usr/local/bin/devvm-secret-scan
```

and all the real logic lives in that one script. When I change it, every repo picks up the change on the next boot. That paid off later, when I switched scanners and no repo needed touching at all.

Here's the script:

```bash
#!/bin/bash
# Run by pre-commit hooks: scans the staged changes for secrets.
set -uo pipefail
if ! command -v betterleaks > /dev/null 2>&1; then
  echo "devvm-secret-scan: betterleaks is not installed, so this commit was NOT scanned for secrets. Re-run ./update-vm.sh on the Mac." >&2
  exit 0
fi
# Hooks run from the repo root. A .betterleaksignore (or .gitleaksignore)
# there is read automatically; a .betterleaks.toml only with --config.
cfg=()
[ -f .betterleaks.toml ] && cfg=(--config .betterleaks.toml)
# --log-level warn: silent on a clean commit; findings still print.
if betterleaks git "${cfg[@]}" --pre-commit --staged --redact --no-banner --verbose --log-level warn; then
  exit 0
fi
{
  echo
  echo "devvm-secret-scan: COMMIT BLOCKED. betterleaks found what looks like a secret in the staged changes (above),"
  echo "  or could not run (a bad .betterleaks.toml, say)."
  echo "  A real secret: unstage it and keep it outside the repo (an environment variable, or a file under ~/.config/)."
  echo "    If it was ever pushed or shared, rotate it."
  echo "  A false positive: add a 'betterleaks:allow' comment on that line, or put the Fingerprint shown above in"
  echo "    .betterleaksignore at the repo root, and commit that file."
} >&2
exit 1
```

A few things in there are deliberate:

- **It's quiet when there's nothing to say.** A hook that prints something on every commit soon gets ignored, so a clean commit prints nothing.
- **It redacts the secret.** The output ends up in agent transcripts and terminal scrollback. A scanner that catches a key and then prints it in full isn't much help.
- **The error message tells you what to do.** I wrote it for the agent as much as for me. When an agent only sees "exit 1", its first idea is often `--no-verify`. When it's told "unstage it and use an environment variable", that's what it does.
- **It lets the commit through if Betterleaks is missing.** If the download fails during provisioning, should every commit in the VM be blocked? I decided no: it warns loudly instead, and the VM's health check fails too, so I notice.

## Making sure it's actually there

Since the scan fails open, I wanted something that would tell me when it wasn't running. The VM is built with Lima, which can run probes at boot, so there's one for this:

```yaml
- description: "Secret scanning (betterleaks)"
  script: |
    #!/bin/bash
    set -eux -o pipefail
    betterleaks version
    test "$(git config --system init.templateDir)" = /usr/local/share/devvm/git-template
    test -x /usr/local/share/devvm/git-template/hooks/pre-commit
  hint: "Commits are not being scanned for secrets. Re-run ./update-vm.sh and check its warnings"
```

The version is pinned in `dev.yaml` with a Renovate comment, so upgrades show up as PRs and don't happen quietly:

```yaml
# renovate: datasource=github-releases depName=betterleaks/betterleaks extractVersion=^v(?<version>.*)$
BETTERLEAKS_VERSION: "1.10.1"
```

## Keeping the agent honest

A hook only works if nobody skips it, and `git commit --no-verify` skips every hook. I can't block that flag, but I can tell Claude not to use it. The user-wide `CLAUDE.md` that the VM writes says:

> Never bypass a git hook with `--no-verify` (or `-n`) unless the user asks for it. If a hook blocks a commit, such as the betterleaks secret scan, fix the cause; if it looks like a false positive, tell the user and ask before adding a `betterleaks:allow` comment or a `.betterleaksignore` entry.

The second half matters as much as the first. Without it, an agent could "fix" a finding by allow-listing the line. I'd rather see every exemption before it goes in.

## Switching to Betterleaks

This week I moved from Gitleaks to Betterleaks. It's the successor from the same maintainers, and because of the shim the switch was small: swap the binary, delete the old `gitleaks` so it doesn't hang around, change the command, and update the allow-comment name in the docs.

One thing caught me out. Gitleaks loads a repo's `.gitleaks.toml` on its own. Betterleaks doesn't.

{{< notice type="warning" >}}
Betterleaks only reads a `.betterleaks.toml` when you pass `--config`, which is why the script checks for one. The format changed too: exemptions are Expr `filter`s, not `[[allowlists]]`. A `.betterleaksignore` (or the old `.gitleaksignore`) is still picked up automatically.
{{< /notice >}}

## What it doesn't cover

It's not airtight, and I'd rather be honest about where the gaps are:

- `--no-verify` still works. The `CLAUDE.md` rule is a request, not a lock.
- A repo with its own `pre-commit` hook isn't scanned unless that hook calls `devvm-secret-scan`. The boot output names those repos, and my `apply-project-standards` skill adds the call when it sets up a repo's hooks.
- The hook only checks new commits. To check a repo's history, I run it once by hand:

  ```bash
  betterleaks git --redact -v   # add --config .betterleaks.toml if the repo has one
  ```

- Commits made on the Mac aren't scanned. All my work happens in the VM, so I can live with that.

And I still turn on GitHub's secret scanning with push protection for every repo (Settings → Advanced Security). It's free for public repos and catches anything that gets past the hook. But by the time GitHub sees a secret, it's already in my local history. The hook catches it before that.
