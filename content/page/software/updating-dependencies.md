---
title: "Keeping Dependencies Up to Date with Renovate"
layout:     page
draft: false
---

## Keeping Dependencies Up to Date with Renovate

### The Problem

Every application sits on a foundation of dependencies — libraries, frameworks, build tools, base images. That foundation is not static. Dependencies release updates continuously: new features, performance improvements, bug fixes, and critically, security patches.

The default human response to this is inertia. Updating dependencies takes time, updates occasionally break things, and there is always something more immediately valuable to work on. So updates get deferred. Weeks become months, months become years, and the gap between what you're running and what's current grows quietly in the background.

The consequences are predictable. A critical security vulnerability is disclosed in a library you're running a version behind — and the fix requires upgrading through three major versions, each with breaking changes. What should have been a routine update becomes a multi-day project with uncertain scope. Or worse, the vulnerability sits unpatched because the upgrade cost is too high to justify right now.

Renovate is a tool that changes the economics of this problem entirely.

---

### What Renovate Does

Renovate is an automated dependency update tool that monitors your repositories and raises pull requests whenever a dependency has a newer version available. It supports an enormous range of package ecosystems — npm, Maven, Gradle, pip, Cargo, Docker, Helm, GitHub Actions, and many more — and can manage all of them within a single repository or across an entire organisation.

The core loop is simple:

1. Renovate scans your dependency files on a schedule
2. It detects newer versions against the relevant registry
3. It opens a pull request with the update applied, including changelog information and links to release notes
4. You review, run your CI pipeline, and merge — or configure Renovate to auto-merge certain classes of update

The PR is the key artifact. It makes dependency updates visible, reviewable, and traceable. Instead of a developer manually bumping a version and committing directly, every update has a record, a CI run, and an explicit merge decision.

---

### The Small and Frequent Principle

The most important benefit of Renovate isn't automation — it's cadence. By updating continuously and in small increments, you stay close to current. And staying close to current makes each individual update low-risk.

Consider the difference:

**Without Renovate:** A library releases 24 patch versions, 4 minor versions, and 2 major versions over 18 months. You update once. You're absorbing 30 releases simultaneously, with no clear signal about what changed when, which change introduced which behaviour, and whether a problem in your test suite was caused by the patch you're looking at or something from six months ago.

**With Renovate:** Each of those releases arrives as a separate PR, with its own changelog, its own CI run, and its own merge decision. Problems are isolated. The blast radius of any single update is tiny. Reverting is straightforward.

This is the same principle that makes continuous integration valuable — small, frequent integration is fundamentally safer than large, infrequent integration. Renovate applies it to the dependency layer.

---

### Security

The security case for Renovate is direct. Many security vulnerabilities exist as a race between disclosure and patching — the vulnerability is known publicly before many systems running the affected version have updated.

Renovate shortens your exposure window. When a security patch is released, a PR appears in your repository quickly — often within hours depending on your schedule configuration. Your CI pipeline runs against it. If it passes, you can merge the same day. Without automation, the same patch might wait weeks for someone to notice it, prioritise it, and find time to do it.

Renovate also integrates with vulnerability databases. With the right configuration it can flag PRs that address known CVEs, raise the priority of security-related updates, and give you visibility into which of your current dependencies have known vulnerabilities — regardless of whether an update is available yet.

For organisations subject to compliance requirements — SOC 2, ISO 27001, PCI DSS — having an auditable, automated process for dependency updates is increasingly expected rather than optional.

---

### Configuration

Renovate is highly configurable through a `renovate.json` file in your repository. A minimal starting configuration might look like:

```json
{
  "extends": ["config:base"],
  "schedule": ["every weekend"],
  "prConcurrentLimit": 5,
  "labels": ["dependencies"]
}
```

This tells Renovate to run on weekends, open no more than five PRs at once, and label all dependency PRs for easy filtering.

**Grouping related updates** reduces noise. Instead of separate PRs for every Spring Boot submodule, group them:

```json
{
  "packageRules": [
    {
      "matchPackagePrefixes": ["org.springframework.boot"],
      "groupName": "Spring Boot"
    }
  ]
}
```

**Auto-merging patch updates** is a common and reasonable policy. Patch versions should be backwards-compatible by definition, and if your test suite passes, the risk of merging automatically is low:

```json
{
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "matchCurrentVersion": "!/^0/",
      "automerge": true
    }
  ]
}
```

Minor and major updates typically warrant human review — minor because they occasionally contain meaningful behaviour changes, major because they often have breaking changes and require more care.

**Pinning vs range updates** is worth considering explicitly. Pinning to exact versions gives you reproducible builds and makes Renovate's PRs precise and reviewable. Ranges give flexibility but obscure exactly what version you're running. For application code, pinning is generally preferable. For libraries you publish, ranges make more sense.

---

### Handling Major Version Upgrades

Even with Renovate, major version upgrades require work. Breaking changes are breaking changes — automation can open the PR, but it can't rewrite your code to adapt to a new API.

What Renovate does is make the upgrade visible and bounded. Rather than discovering you're four major versions behind when a vulnerability forces your hand, you see each major version PR as it becomes available. You can schedule the work deliberately, understand the scope before you start, and spread the effort over time.

Renovate also surfaces the release notes and changelog directly in the PR description — so the context you need to assess the upgrade is in the same place as the code change.

---

### Organisational Adoption

Renovate can be run as a self-hosted tool or via the Renovate cloud app (Mend Renovate) which connects directly to GitHub, GitLab, or Bitbucket. For organisations with many repositories, the cloud app provides a dashboard showing dependency update status across all of them — giving engineering leadership visibility into how current the fleet actually is.

A common adoption pattern:

1. **Start with one repository** — enable Renovate, observe the volume of PRs, tune the configuration for noise and grouping
2. **Establish a merge habit** — agree that dependency PRs get reviewed and merged on a cadence (weekly is common), not left to accumulate
3. **Enable auto-merge for patch updates** — once confidence in the CI pipeline is established
4. **Roll out across repositories** — using a shared `renovate.json` preset so configuration is consistent and centrally maintained

The merge habit is as important as the tooling. Renovate opening PRs that nobody merges recreates the same backlog problem in a different form. The organisational commitment is to treat dependency updates as routine maintenance — small, expected, and regularly completed — rather than a project that gets scheduled when the pain becomes acute.

---

### The Broader Principle

Renovate is a specific tool for a specific problem, but it embodies a wider principle that runs through good software practice: **make the right thing easy and the wrong thing visible**.

Falling behind on dependencies is easy when there's no automation — it requires no action, it happens by default. Renovate inverts this. Staying current becomes the path of least resistance. Falling behind becomes visible — the PRs accumulate, the dashboard turns amber, the signal is clear.

This is the same principle behind CI breaking the build on a failing test, or a linter flagging a code smell inline. The feedback is immediate, the cost of addressing it is low, and the alternative — letting it accumulate — becomes the harder path.
