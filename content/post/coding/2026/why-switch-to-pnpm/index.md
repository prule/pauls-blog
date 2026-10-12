---
layout: post
title: "Why You Should Switch to pnpm"
description: "Stop wasting disk space and CI minutes—pnpm uses a content-addressable store to make node_modules fast, strict, and efficient."
date: 2026-06-08T20:58:00+10:00
author: "Paul"
publishDate: 2026-06-08
tags:
  - NodeJS
  - pnpm
  - Performance
  - Productivity
categories: [ coding ]
---

If you are still using npm or Yarn Classic, you are wasting gigabytes of local storage, slowing down your CI/CD pipelines, and opening the door to silent dependency bugs. 

**pnpm** (performant npm) solves these issues out of the box. By replacing file duplication with a single, content-addressable store and using hard links to assemble `node_modules`, pnpm makes dependency resolution fast, strict, and disk-efficient.

<!--more-->

## The Core Problem: The node_modules Black Hole

Traditional package managers have a fundamental flaw: they duplicate files. If you have ten projects using React, `npm` will download and save React ten times on your hard drive. This wastes disk space and slows down installation times since every dependency must be copied file-by-file into each project's directory.

Additionally, to resolve nested dependencies without deep directory trees, npm flattens `node_modules`. This flattening introduces **phantom dependencies**—where your application can successfully import a package that is not declared in your `package.json`, simply because another package you installed happens to depend on it.

## How pnpm Works: Content-Addressable Storage

Instead of duplicating files or flattening directories, pnpm keeps a single, global content-addressable store at `~/.local/share/pnpm/store`.

When you run `pnpm install`, pnpm checks the store first. If a package has already been downloaded, pnpm creates a **hard link** from the global store to your project's `node_modules`. 

{{< mermaid >}}
graph LR
%% Global Store Styling
classDef store fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,stroke-dasharray: 5 5;
%% Project Styling
classDef project fill:#f9fbe7,stroke:#689f38,stroke-width:2px;
%% Link Styling
classDef linkNode fill:#fff3e0,stroke:#f57c00,stroke-width:1px;

    subgraph GlobalStore [Global Content-Addressable Store]
        pkgA["lodash@4.17.21 (.js/.json files)"]
        pkgB["react@18.2.0 (.js/.json files)"]
    end
    class pkgA,pkgB store;

    subgraph ProjectA [Project A .pnpm virtual store]
        pA_sym["node_modules/lodash"] -- Symlink --> pA_hard["nested lodash@4.17.21/node_modules/lodash"]
        pA_deps["node_modules (Project Root)"] -- Symlink --> pA_sym
    end
    class pA_sym,pA_hard,pA_deps project;

    subgraph ProjectB [Project B .pnpm virtual store]
        pB_sym["node_modules/lodash"] -- Symlink --> pB_hard["nested lodash@4.17.21/node_modules/lodash"]
        pB_deps["node_modules (Project Root)"] -- Symlink --> pB_sym
    end
    class pB_sym,pB_hard,pB_deps project;

    %% Cross-subgraph Hard Links to Global Store
    pA_hard --> |Hard link| pkgA
    pA_deps --> |Hard link to direct dep| pkgB
    pB_hard --> |Hard link| pkgA

{{< /mermaid >}}

Because it uses hard links, your project directories point to the exact same physical sectors on your disk. Installing a package in a second project takes milliseconds because no new files are written to disk.

## Three Key Advantages

### 1. Massive Disk Space Savings
With pnpm, your dependencies take up space exactly once. If different projects use different versions of the same package, only the files that changed between versions are added to the store. If they use the exact same version, they share the same physical storage.

### 2. Elimination of Phantom Dependencies
pnpm creates a strict, nested `node_modules` layout. It only symlinks packages explicitly declared in your `package.json` into the root of your project's `node_modules`. 

Transitive dependencies are nested inside `node_modules/.pnpm`, keeping them hidden from Node’s module resolution algorithm. If you try to import a package that isn't in your `package.json`, Node will fail immediately. This prevents fragile code that breaks unpredictably when sub-dependencies update.

{{< notice type="warning" title="Real-World Gotcha: Vite/Rollup Build Failures" >}}
Because pnpm blocks phantom dependencies, tools that dynamically generate virtual modules (like `vite-plugin-pwa`) may fail during build with errors like:
```text
Rollup failed to resolve import "workbox-window" from "/@vite-plugin-pwa/virtual:pwa-register/vue"
```
This happens because the virtual module expects `workbox-window` to be hoisted to the root of `node_modules`. To fix it, you must explicitly declare the sub-dependency in your own project:
```bash
pnpm add -D workbox-window
```
{{< /notice >}}


### 3. Supercharged CI/CD Caching
Because pnpm keeps its global store isolated, caching in CI environments is incredibly simple and highly effective. In GitHub Actions, you can cache the pnpm store directly, reducing clean install times by up to 80%.

Here is a standard workflow utilizing `actions/setup-node`’s native pnpm cache integration:

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run Lint
        run: pnpm lint

      - name: Run Format Check
        run: pnpm format:check

      - name: Run Type Check
        run: pnpm typecheck

      - name: Run Build
        run: pnpm build
        
      - name: Run tests
        run: pnpm test
```

## How to Migrate to pnpm

Switching an existing project is straightforward and takes less than a minute.

1. **Install pnpm:**
   Avoid installing `pnpm` globally via `npm` (e.g., `npm install -g pnpm`), as this ties it to a single Node version and causes it to disappear if you switch Node versions. Instead, use a standalone installation.

   Using the official standalone script:
   ```bash
   curl -fsSL https://get.pnpm.io/install.sh | sh -
   ```

   Using Homebrew (macOS/Linux):
   ```bash
   brew install pnpm
   ```

2. **Import your current lockfile:**
   Generate a `pnpm-lock.yaml` file from your existing `package-lock.json` or `yarn.lock` to lock down the exact same dependency tree:
   ```bash
   pnpm import
   ```

3. **Clean up old artifacts:**
   Remove the old `node_modules` directory and lockfile:
   ```bash
   rm -rf node_modules package-lock.json yarn.lock
   ```

4. **Perform a clean install:**
   ```bash
   pnpm install
   ```

{{< notice type="tip" title="Handling Peer Dependency Warnings" >}}
Because pnpm enforces strict dependency resolution, you might see warning logs about missing peer dependencies that npm previously ignored silently. You can configure peer dependency rules in your package.json under `pnpm.peerDependencyRules` or configure `.npmrc` to auto-install peers if required.
{{< /notice >}}

{{< notice type="warning" title="Security Update: Ignored Build Scripts in pnpm v11+" >}}
Starting with **pnpm v11**, build and install scripts of dependencies (such as native binaries like `esbuild`) are blocked by default for security. 

If you get a `[ERR_PNPM_IGNORED_BUILDS]` error, you must explicitly allow them. In a workspace/monorepo, you must define this in your `pnpm-workspace.yaml` (not `package.json`):

```yaml
# pnpm-workspace.yaml
allowBuilds:
  esbuild: true
```
{{< /notice >}}

{{< notice type="note" title="Vite Pre-bundling Cache Out of Sync" >}}
If you encounter a `Vite: The file does not exist ... in the optimize deps directory` error after configuring build scripts or upgrading dependencies, Vite's internal cache is likely out of sync.

Fix it by forcing Vite to rebuild its cache:
```bash
pnpm run dev --force
```
Or manually clear the cache folder:
```bash
rm -rf node_modules/.vite
```
{{< /notice >}}

## Making It Stick: Stopping npm From Creeping Back

Migrating is the easy part. The harder part is that `npm install` still works in your pnpm project, and the day someone (or some agent, or you in a hurry) runs it, you get a `package-lock.json`, a flat `node_modules`, and a dependency tree that no longer matches what CI resolves.

No single guard catches every route in, so layer them. Each one below catches what the previous one misses.

### Layer 1: `only-allow` as a preinstall hook

```json
"scripts": { "preinstall": "npx only-allow pnpm" }
```

[`only-allow`](https://github.com/pnpm/only-allow) is pnpm's own tool. It reads the `npm_config_user_agent` environment variable to detect which package manager invoked it, and exits non-zero with a readable message if it's the wrong one. This blocks both `npm install` and `yarn install`.

Two gaps worth knowing: `npm install --ignore-scripts` skips `preinstall` entirely, and on a fresh clone `npx` has to fetch `only-allow` over the network before it can run — it cannot be a devDependency, because `preinstall` runs *before* dependencies are installed.

### Layer 2: Refuse to commit the competing lockfiles

```gitignore
package-lock.json
yarn.lock
npm-shrinkwrap.json
```

This is the layer that actually holds, because it doesn't rely on a script running. An accidental `npm install` still produces a local `node_modules`, but its lockfile can never be committed — so the mistake cannot spread to CI or to another machine.

### Layer 3: Fail CI if one shows up anyway

```yaml
name: package-manager
on: [push, pull_request]
jobs:
  lockfile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Reject non-pnpm lockfiles
        run: |
          fail=0
          for f in package-lock.json yarn.lock npm-shrinkwrap.json; do
            if [ -f "$f" ]; then
              echo "::error file=$f::This project uses pnpm. Remove $f and run 'pnpm install'."
              fail=1
            fi
          done
          if [ ! -f pnpm-lock.yaml ]; then
            echo "::error::pnpm-lock.yaml is missing — it must be committed."
            fail=1
          fi
          exit $fail
```

Catches both the `--ignore-scripts` bypass and a lockfile that got force-added past `.gitignore`. It also catches the opposite mistake — someone deleting `pnpm-lock.yaml` and not committing the replacement.

### Layer 4: A shell guard, if you control the environment

The three layers above all catch the mistake *after* it happens. If you have a single dev box or VM, you can catch it before anything is written:

```bash
npm() {
  case "$1" in
    install|i|ci|add|update)
      if [ -f pnpm-lock.yaml ]; then
        echo "pnpm-lock.yaml present — use pnpm, not npm" >&2; return 1
      fi ;;
  esac
  command npm "$@"
}
```

Scoped to install-type commands, and only when a `pnpm-lock.yaml` is actually present — so `npm install -g` keeps working, and so do any projects you have that legitimately still use npm.

{{< notice type="tip" title="What about engine-strict and Corepack?" >}}
Two other approaches get recommended often, and I'd skip both.

Setting `"engines": { "npm": "please-use-pnpm" }` with `engine-strict=true` in `.npmrc` makes npm refuse to run. But pnpm reads `engine-strict` from `.npmrc` too, so you're relying on it to ignore an `npm` engine entry it can see — verify that before putting it in a dozen repos.

**Corepack** with `COREPACK_ENABLE_STRICT=1` genuinely does block a mismatched package manager. But Corepack ties your package manager to a Node install, which is the exact problem that standalone pnpm solves — an fnm patch bump shouldn't be able to lose your pnpm. Not worth undoing that for this.
{{< /notice >}}

### One more thing to check: duplicate lockfiles you already have

Before adding guards, look for projects that already have *both*. I found one of mine carrying a `pnpm-lock.yaml` and a `package-lock.json` committed side by side — two different dependency graphs, with which one wins depending on the tool that happens to run. The git log settled it: the pnpm lockfile had been updated recently, the npm one hadn't been touched since the initial commit.

```bash
# find the conflicts
for d in */; do
  [ -f "$d/pnpm-lock.yaml" ] && [ -f "$d/package-lock.json" ] && echo "conflict: $d"
done
```

## Conclusion

Switching to pnpm is a low-risk, high-return upgrade for any development workflow. It frees up storage on your workstation, guarantees that your dependency imports are safe and declared, and speeds up your builds. 

---

*Are you using pnpm or still stuck with traditional node_modules? Let me know on [LinkedIn](https://www.linkedin.com/in/paulrule/)!*
