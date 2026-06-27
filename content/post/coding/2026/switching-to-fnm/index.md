---
title: "Switching from nvm to fnm"
date: 2026-06-26T20:28:52+10:00
publishDate: 2026-06-26
draft: false
layout: "post"
tags: ["node", "fnm", "nvm", "shell", "performance"]
categories: ["coding"]
description: "Drop nvm and switch to fnm to instantly eliminate shell startup lag when managing Node versions."
author: "Paul"
---

Drop `nvm` and install `fnm` (Fast Node Manager). Your shell will open instantly instead of hanging for half a second waiting for Node versions to initialize. 

`fnm` is built in Rust. It's orders of magnitude faster than `nvm`, cross-platform, and fully compatible with existing `.nvmrc` and `.node-version` files.

## Why Switch?

`nvm` is a massive shell script. It sources itself every time you open a new terminal tab. This overhead becomes noticeable, often adding 300-500ms to shell startup times.

`fnm` solves this. It runs as a compiled binary. Shell startup impact is virtually zero. It supports Windows, macOS, and Linux out of the box.

## Removing nvm

First, tear out `nvm`. Delete the installation directory:

```bash
rm -rf ~/.nvm
```

Open your shell configuration file (`~/.zshrc` or `~/.bashrc`) and delete the `nvm` initialization block. It typically looks like this:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

Restart your terminal or run `source ~/.zshrc`.

## Installing fnm

Install `fnm` via its bash script or Homebrew.

Using the bash script:

```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

Using Homebrew (macOS/Linux):

```bash
brew install fnm
```

Hook `fnm` into your shell. Add the following to your `~/.zshrc` or `~/.bashrc`:

```bash
eval "$(fnm env --use-on-cd --resolve-engines --version-file-strategy=recursive)"
```

{{< notice type="info" >}}
- `--use-on-cd`: Automatically switches Node versions when entering a directory with an `.nvmrc` or `.node-version` file.
- `--resolve-engines`: Allows reading the Node version from the `engines` field in `package.json`.
- `--version-file-strategy=recursive`: Searches up the directory tree to find a version file if one isn't present in the current directory.
{{< /notice >}}

Restart your terminal.

## .node-version vs .nvmrc

If you are setting up a new project, prefer using a `.node-version` file instead of `.nvmrc`. 

While `fnm` supports both, `.nvmrc` implies the usage of `nvm`. `.node-version` is a generic standard supported by most modern version managers (including `fnm`, `nodenv`, `asdf`, and `volta`). It's a cleaner approach that signals to other developers that any compliant version manager can be used.

## Reading from package.json

Because we enabled the `--resolve-engines` flag, `fnm` can automatically detect the required Node version from your `package.json` if you don't have a `.node-version` or `.nvmrc` file. 

To use this, add the `engines` field to your `package.json`:

```json
{
  "engines": {
    "node": ">=22.0.0"
  }
}
```

When you `cd` into this directory, `fnm` will parse the `engines.node` range and automatically switch to an installed version that satisfies the requirement.

## Managing Versions

Specifying and switching versions in `fnm` is nearly identical to `nvm`:

```bash
fnm install 22       # Install a specific version
fnm install --lts    # Install the latest LTS release
fnm use 22           # Switch to a specific installed version
fnm default 22       # Set the default version for new shells
node -v
```

Same functionality, zero lag.

## GitHub Actions CI

When setting up your CI pipeline with GitHub Actions, the official `actions/setup-node` step seamlessly supports `.node-version`, `.nvmrc`, and `package.json` out of the box. 

Just point the `node-version-file` parameter to your source of truth:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version-file: '.node-version' # or '.nvmrc' or 'package.json'
```

This keeps your CI pipeline perfectly synced with your local `fnm` environment, avoiding the headache of updating Node versions in multiple places.

## Resources

- [fnm GitHub Repository](https://github.com/Schniz/fnm)
