---
name: hugo-blog-post
description: >
  Writes a blog post in Markdown format for the Hugo static site generator.
  Use this skill whenever the user asks to "write a blog post", "draft a post",
  "create a Hugo post", or wants to produce technical writing for their blog.
  Triggers on any request to produce a blog post, article, or similar long-form
  technical content intended for a Hugo site. Output is a properly formatted
  .md file with Hugo-compatible YAML front matter.
---
 
# Hugo Blog Post Skill
 
Produces a complete Hugo-compatible Markdown blog post saved to a `.md` file.
 
---
 
## Front Matter
 
Always YAML. Always include these fields, in this order:
 
```yaml
---
title: "Post Title"
date: YYYY-MM-DDTHH:MM:SS+10:00          # Sydney timezone
publishDate: YYYY-MM-DD                  # date only, same day as date
draft: true                               # default unless user says otherwise
layout: "post"
tags: ["tag1", "tag2"]
categories: ["category"]
description: "One sentence. What the post covers and why it matters."
author: "Paul"
---
```
 
- `draft`: default `true`
- `layout`: always `"post"`
- `author`: always `"Paul"` unless told otherwise
- `date`: use current date/time in ISO 8601 with Sydney offset (`+10:00` AEST or `+11:00` AEDT)
- `publishDate`: date only (`YYYY-MM-DD`), same day as `date`
- `description`: one tight sentence — no "In this post we will..."
---
 
## Voice & Tone
 
Default: **pragmatic engineer**
- Terse, peer-to-peer, no fluff
- No pleasantries, no hedging, no filler words
- Fragments OK. Short synonyms preferred.
- Write for an experienced developer — skip the basics unless the post is about the basics
- Override only if user explicitly requests different tone
---
 
## Structure
 
Default: **McKinsey Pyramid**
1. **Lead with the answer** — state the conclusion/recommendation up front
2. **Support** — evidence, reasoning, code examples
3. **Context** — background only as needed, after the point is made
Override only if user explicitly requests different structure.
 
---
 
## Shortcodes
 
Use these Hugo shortcodes where appropriate:
 
```
{{< center >}}content{{< /center >}}       # centre-align content
{{< mermaid >}}diagram{{< /mermaid >}}     # Mermaid diagrams
{{< notice type="info" >}}text{{< /notice >}}  # callout boxes (type: info, warning, tip, note)
```
 
Use `mermaid` shortcode (not fenced code blocks) for any diagrams.
Use `notice` for important callouts, warnings, or tips.
 
---
 
## Code Blocks
 
Use fenced code blocks with language identifiers:
 
```
```kotlin
// code here
```
```
 
---
 
## Output
 
1. Derive a directory name from the title: lowercase, hyphens, no special chars (e.g., `my-post-title`).
2. Save the post as `index.md` inside a nested directory structure based on the category and year of publication: `content/posts/<category>/<yyyy>/<title>/index.md` (e.g., `/Users/paulrule/WebstormProjects/pauls-blog/content/posts/coding/2026/my-post-title/index.md`).
   - If the repository uses `content/post/` instead of `content/posts/`, adapt the prefix to `content/post/...` accordingly.
   - If the category is not specified, default to using the `coding` category directory.
3. Present the saved file path clearly in the chat so the user can easily open it.
4. Show the front matter in chat so the user can verify fields at a glance.
 
---
 
## Workflow
 
1. If user provides a topic/outline — write the post
2. If user provides rough notes or bullet points — expand into full post following voice + structure
3. If topic is vague — ask one clarifying question before writing
4. After writing, check:
   - Front matter complete and valid?
   - Lead paragraph states the conclusion?
   - No fluff sentences?
   - Code blocks have language tags?
   - File saved and presented?
