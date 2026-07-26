---
name: update-diary-post
description: >
  Updates Paul's monthly diary posts in Markdown format with curated links, videos,
  podcasts, posts, and tools. Automatically searches the web for descriptions and titles
  when only URLs are provided, categorizes them, formats them, and writes them directly
  into the diary Markdown file (e.g. content/post/diary/2026/2026-07.md). Creates new
  monthly diary files if they do not yet exist.
---

# Update Diary Post Skill

Appends new content (videos, podcasts, blog posts, books, tools) to Paul's monthly diary posts. 

---

## 1. Locate the Target File

1. Check the current local date.
2. Formulate the month: `YYYY-MM` (e.g., `2026-07`).
3. Locate the diary file at `content/post/diary/<YYYY>/<YYYY>-<MM>.md` (e.g., `content/post/diary/2026/2026-07.md`).
4. If the file does not exist, initialize a new diary post for that month (see the [New File Initialization](#new-file-initialization) section below).

---

## 2. Research & Curate Links

For any links provided by the user without titles or descriptions:
1. Use `search_web` to retrieve the real title and a summary of key points.
2. Group the items into one of these four sections:
   * **## Videos**: YouTube videos, tech keynotes, documentaries.
   * **## Podcasts**: Audio podcasts, episodes, interview recordings.
   * **## Posts**: Blog articles, research papers, newsletters, security advisories, LinkedIn/Twitter posts.
   * **## Other**: Developer tools, software repositories, websites, books, miscellaneous resources.

---

## 3. Formatting Rules

Format all entries consistently with existing items:
- Use `* [Title](URL)` for the main link.
- Use `  + Description` (indented by 2 spaces, with a plus sign) for the description bullets.
- Keep descriptions crisp, direct, and written in a pragmatic engineering tone.

Example:
```markdown
* [Refactoring.Guru](https://refactoring.guru/)
  + A beautifully designed, beginner-friendly resource providing comprehensive guides on design patterns, code smells, and refactoring techniques.
```

---

## 4. Writing to the File

1. Read the existing file using `view_file` to determine line positions.
2. Use editing tools (`replace_file_content` or `multi_replace_file_content`) to append new items to the bottom of their respective sections:
   - Ensure you do not destroy existing list items.
   - Maintain a single empty line between list items or before the next heading where appropriate.
3. Save the changes and present the diff/changes clearly to the user.

---

## New File Initialization

If the diary post for the current month does not exist, initialize it with the following template structure:

```markdown
---
layout:     post
title: "Diary YYYY-MM"
description: "Diary YYYY-MM"
date:    YYYY-MM-DD
author: "Paul"
publishDate: YYYY-MM-01
tags:
  - diary
  - podcasts
  - videos
  - posts
categories: [ diary ]
draft: false
---

# Month YYYY

Things I'm seeing/reading/watching/listening to in Month YYYY...

## Videos

## Podcasts

## Posts

## Other
```

- Replace `YYYY`, `MM`, `DD`, `Month` with the actual values for the current local time.
- Set `publishDate` to the first day of that month.
- Set `date` to the current local date.
