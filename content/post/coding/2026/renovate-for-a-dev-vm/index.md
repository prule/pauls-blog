---
title: "Letting Renovate Upgrade My Dev VM"
date: 2026-11-25T15:20:00+11:00
publishDate: 2026-11-25
draft: false
layout: "post"
tags: ["renovate", "lima", "devvm", "dependencies", "yaml", "regex"]
categories: ["coding"]
description: "How one regex custom manager lets Renovate open PRs for every tool version pinned in my dev VM's Lima config, which bumps wait for approval, and the pins it can't reach."
author: "Paul"
---

My [dev VM](https://github.com/prule/devvm) is defined by one file, `dev.yaml`. It pins the version of every tool the VM installs: Node, pnpm, OpenSpec, the Supabase CLI, SonarQube, Postgres, betterleaks, hadolint and more. Pinning is the right call, because every project uses the same versions and a rebuild gives me the same machine. But pins go stale, and I don't want to check a dozen release pages by hand.

So Renovate does it. Each pin has a comment above it saying where its releases come from, one regex in `renovate.json` matches them all, and a new release turns up as a PR against `dev.yaml`.

```yaml
# renovate: datasource=npm depName=pnpm
PNPM_VERSION: "12.21.0"
# renovate: datasource=github-releases depName=supabase/cli extractVersion=^v(?<version>.*)$
SUPABASE_CLI_VERSION: "2.122.1"
# renovate: datasource=docker depName=postgres
POSTGRES_VERSION: "18.7"
```

## Why Renovate needs help here

Renovate understands `package.json`, `build.gradle.kts`, Dockerfiles and a long list of other formats. It doesn't understand a Lima config, and there's no reason it should. In `dev.yaml` the versions are just strings in a `param:` block, which the provisioning scripts read as `$PARAM_PNPM_VERSION` and so on. Nothing about `PNPM_VERSION: "12.21.0"` says "this is pnpm, from npm".

That's what the comment adds. It names the [datasource](https://docs.renovatebot.com/modules/datasource/) (npm, Docker, GitHub releases, PyPI, Node versions) and the package. Then a custom manager teaches Renovate to read it.

## The custom manager

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "customManagers": [
    {
      "description": "Toolchain pins in dev.yaml, each annotated with a `# renovate:` comment",
      "customType": "regex",
      "managerFilePatterns": ["/^dev\\.yaml$/"],
      "matchStrings": [
        "# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+)( extractVersion=(?<extractVersion>\\S+))?\\s+\\w+: \"(?<currentValue>[^\"]+)\""
      ]
    }
  ]
}
```

One regex covers every pin. The named groups are what Renovate looks for: `datasource`, `depName` and `currentValue`, plus an optional `extractVersion`. That last one handles release tags with a prefix. GitHub releases are usually tagged `v2.122.1`, while the pin holds `2.122.1`, so `extractVersion=^v(?<version>.*)$` strips the `v`.

Adding a new tool is two lines: the pin and its comment. No change to `renovate.json`.

{{< notice type="warning" >}}
The comment has to be on the line **directly** above the value. The regex allows only whitespace between them. Put another comment in between, and Renovate silently stops tracking that pin. Nothing errors; the pin just stops getting updates. `dev.yaml` carries a comment saying exactly that, next to the pins.
{{< /notice >}}

## Majors wait for me

Most bumps are minor or patch releases, and Renovate opens those PRs on its own. Majors are different. A major Node or pnpm bump changes every project in the VM at once, so it should be a decision, not something that turns up in a PR:

```json
"packageRules": [
  {
    "description": "A major bump changes every project at once; open it only when asked from the Dependency Dashboard",
    "matchFileNames": ["dev.yaml"],
    "matchUpdateTypes": ["major"],
    "dependencyDashboardApproval": true
  }
]
```

With `dependencyDashboardApproval`, a major shows up as a checkbox on Renovate's Dependency Dashboard issue, and the PR opens only once I tick it.

For Postgres this matters even more. Postgres can't open a data directory from an older major, so a major bump means a rebuild or a dump and restore. The same pin also sets the major version of the `psql` client the VM installs, so the server and client move together.

## What it can't reach

**Java.** The VM gets its JDKs from SDKMAN, and the pins are SDKMAN identifiers:

```yaml
JAVA_VERSIONS: "25.0.4-tem 21.0.12-tem"
JAVA_DEFAULT: "25.0.4-tem"
```

Renovate has no SDKMAN datasource, so these get no comment and I bump them by hand.

**Hugo.** This one is deliberately not tracked in `dev.yaml`. The version that counts is the one my blog deploys with, `HUGO_VERSION` in the blog repo's GitHub Actions workflow. Renovate bumps it there, and the VM installs whatever that workflow says, so my local preview builds with the same Hugo as the live site. `dev.yaml` still has a `HUGO_VERSION`, but only as a fallback for before the blog is cloned. It has no Renovate comment, because keeping it current would be pointless.

That's a useful rule in general: a version should be pinned in one place. Where the thing that really matters is somewhere else, point at it rather than keep a second copy in step.

## From PR to VM

Merging a Renovate PR changes `dev.yaml` on GitHub, and that's all. The running VM doesn't notice. The new version arrives the next time I apply `dev.yaml` to the VM, which pulls the latest file, restarts the VM and re-runs provisioning. The install steps compare the installed version with the pin and only download when they differ.

So the flow is: Renovate opens a PR, I read the release notes and merge it, and the next update brings the VM up to date. Every tool upgrade is now a reviewed commit in its history.
