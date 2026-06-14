---
title: "Rebuilding a Simpler, Local-First Video Aggregator"
date: 2026-06-10T14:11:22+10:00
publishDate: 2026-06-10
draft: false
layout: "post"
tags: ["Web dev", "PWA", "IndexedDB", "Cloudflare"]
categories: ["coding"]
description: "Rebuilding a YouTube feed aggregator as a streamlined, local-first PWA deployed on Cloudflare Pages."
author: "Paul"
---

I rebuilt my YouTube content aggregator and filter ([video.vamonossoftware.com](https://video.vamonossoftware.com)) as a streamlined, local-first Progressive Web App (PWA). By stripping out over-engineered features like local LLM tagging and shifting to a client-side IndexedDB model, the new version establishes a fast, responsive, and maintainable foundation.

## The Architecture

The application operates entirely client-side, eliminating remote database dependencies and hosting costs. 

{{< mermaid >}}
graph TD
    A[Remote Source URL / GitHub] -->|Import & Sync| B[Sources Config]
    C[YouTube API Key] -.->|Direct API Query| D[YouTube Feed Data]
    E[Cloudflare Worker RSS Proxy] -.->|Fallback Fetch| D
    B --> D
    D --> F[Filter Rules: Include / Exclude]
    F --> G[Topics Feed: Motorsport, Tennis, etc.]
    G --> H[(Browser IndexedDB)]
{{< /mermaid >}}

### 1. Local-First Storage
All feed data, subscription lists, topics, and filtering rules are stored directly in the browser using **IndexedDB**. This keeps interactions instantaneous and respects user privacy.

### 2. Hybrid Feed Fetching
To bypass CORS restrictions and API limits, the aggregator uses a two-tiered fetching strategy:
* **API Key Mode**: If you provide a YouTube API key in the settings, the app queries YouTube's data API directly from the browser.
* **RSS Fallback**: Without an API key, the app requests YouTube channel RSS feeds through a lightweight proxy deployed as a Cloudflare Worker.

### 3. Shareable Sources & Topic Filters
Subscriptions are structured under **Sources**. In addition to local configuration, sources can be imported and synced from remote URLs (e.g., a raw JSON file hosted in a GitHub repository). This allows users to curate and share subscription feeds. 

Customizable Include/Exclude rules then process these feeds, sorting videos chronologically under user-defined **Topics** (such as *Motorsport* or *Tennis*).

---

## Lessons in Simplicity

The first iteration of this aggregator was overbuilt. It attempted to auto-tag videos using a local LLM, introducing unnecessary latency, configuration friction, and complexity. 

By replacing the LLM with straightforward string-matching filters (Include/Exclude rules), the app became faster and easier to maintain. This simpler foundation is more reliable and makes it easier to identify which features are actually worth adding next.

---

## The AI-Assisted Build Stack

Developing this iteration was an exercise in high-velocity prototyping using modern AI tooling:
* **Claude Design & Claude Code**: Handled UI/UX generation, code refactoring, and test suite execution.
* **OpenSpec**: Streamlined structural specification.
* **Cloudflare Pages**: Auto-deploys the build on merge to `main`, keeping the deployment loop completely hands-off.

The result is a clean, dependency-light codebase that delivers a predictable, algorithm-free YouTube feed.

Still very much in the alpha phase where I'm trying to figure out how it should work for the best user experience. Use at your own risk...
