---
title: "Turning a Markdown Repository into a Hosted Docs Site with MkDocs"
date: 2026-06-28T22:14:08+10:00
publishDate: 2026-06-28
draft: true
layout: "post"
tags: ["mkdocs", "documentation", "github-actions"]
categories: ["coding"]
description: "How to convert a flat folder of Markdown files into a beautiful, searchable documentation site using MkDocs and GitHub Actions."
author: "Paul"
---

Convert flat Markdown files into a searchable MkDocs site by moving them into a `docs/` folder, adding a simple YAML config, and deploying via GitHub Actions.

{{< notice type="info" >}}
While often referred to colloquially as a "GitBook", we use MkDocs with the Material theme because it's open-source, highly customizable, and looks incredibly professional out of the box.
{{< /notice >}}

### 1. Reorganize the Files

Static site generators expect a specific directory structure. Move all your standalone `.md` files into a dedicated `docs/` folder. Your root `ReadMe.md` should be copied to `docs/index.md` to serve as the homepage.

```bash
mkdir -p docs
# Move all markdown files except ReadMe.md
for file in *.md; do if [ "$file" != "ReadMe.md" ]; then mv "$file" docs/; fi; done
# Copy ReadMe to act as the homepage
cp ReadMe.md docs/index.md
```

### 2. Configure MkDocs

Create an `mkdocs.yml` file in the root of your project. This file defines the site name, theme, and the sidebar navigation structure.

```yaml
site_name: The Engineering Playbook
theme:
  name: material
  palette:
    - scheme: default
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    - scheme: slate
      toggle:
        icon: material/brightness-4
        name: Switch to light mode

nav:
  - Introduction: index.md
  - Architecture:
    - Clean Architecture: CleanArchitecture.md
```

The `nav` array maps the sidebar labels to your Markdown files.

### 3. Deploy via GitHub Actions

To host the site for free on GitHub Pages, add a workflow file at `.github/workflows/docs-deploy.yml`. 

```yaml
name: Deploy docs
on:
  push:
    branches:
      - main
permissions:
  contents: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: 3.x
      - run: pip install mkdocs-material 
      - run: mkdocs gh-deploy --force
```

When you push to `main`, this workflow installs MkDocs, builds the static HTML, and forcefully pushes it to the `gh-pages` branch. GitHub Pages will then automatically serve your beautiful new documentation site.

### 4. Improving Developer Experience (The `./run` Script)

To make it trivial for your team to write documentation locally without memorizing Python commands, add a simple executable shell script named `run` to the root of your project.

```bash
#!/usr/bin/env bash
set -e

case "$1" in
    setup)
        echo "Installing MkDocs..."
        pip install mkdocs-material
        ;;
    serve)
        echo "Starting local server..."
        mkdocs serve
        ;;
    build)
        echo "Building static site..."
        mkdocs build
        ;;
    help|*)
        echo "Usage: ./run [setup|serve|build]"
        ;;
esac
```

Now, anyone pulling the repository can simply type `./run setup` followed by `./run serve` to instantly spin up a live-reloading local preview of the documentation!
