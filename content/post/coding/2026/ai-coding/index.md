---
layout:     post
title: "AI-Assisted Engineering: From Implementation to Specification"
description: "Are we using it properly"
date:    2026-04-20
author: "Paul"
publishDate: 2026-04-20
tags:
  - Programming
  - AI
  - Productivity
categories: [ code ]
draft: false
---

The more I use AI to assist with coding, the more I realize that **context is the primary currency of effective assistance.** The quality of the output is directly proportional to the quality of information available to the assistant.

This realization has fundamentally changed how I work. I’ve started treating ADRs (Architecture Decision Records) and design documents not as "after-the-fact" documentation, but as essential inputs to be versioned in Git alongside the code. Goodbye Confluence; hello documentation-as-code.

### The Shift to Specification-Driven Development

There is a growing epiphany in the industry: the better we document the features we want, the better results we get from AI. But this isn't just about "better prompts." It’s about a shift toward **Spec-Driven Development**.

When we spend more time figuring out exactly what we want and documenting it clearly, implementation time shrinks. This is true whether a human or an AI is doing the work, but AI has made the ROI of a good specification much more immediate.

I recently encountered the **BMAD** (Business, Model, Action, Data) framework via the [Tech Lead Journal](https://podcasts.apple.com/au/podcast/tech-lead-journal/id1523421550?i=1000762391704), and it reinforced that many of us (myself included) aren't yet using these tools to their full potential. We are still learning how to move from "writing code" to "orchestrating intent."

### The Emerging Ecosystem

This isn't just a change in mindset; a massive ecosystem of tools and playbooks is emerging to solve the "orchestration" problem:

*   **Agent Skills & Playbooks:** [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) and the [AI Engineering Playbook](https://github.com/lilacmohr/ai-engineering-playbook).
*   **Structured Automation:** Tools like [get-shit-done](https://github.com/gsd-build/get-shit-done), [superpowers](https://github.com/obra/superpowers), and [ai-assisted-engineering](https://github.com/JohnCrickett/ai-assisted-engineering).
*   **Agent Frameworks:** [agency-agents](https://github.com/msitarzewski/agency-agents).

### Incrementalism vs. Waterfall

There is a risk that "Spec-Driven Development" sounds like a return to Big Design Up Front (Waterfall). However, I believe the future is **Incremental Specification**. We should still learn and verify as we go, using the AI to build thin, vertical slices of a feature, verifying the result, and then refining the spec for the next slice.

Even [DHH has noted this shift](https://podcasts.apple.com/au/podcast/the-pragmatic-engineer/id1769051199?i=1000760299204), moving from AI-skepticism to seeing it as a transformative tool for the pragmatic programmer.

### The Path Forward

We aren't "there" yet. The tools and our mental models are still evolving—significantly so over just the last three years. But the direction is clear: the most valuable skill for a developer is no longer just "knowing the syntax," but the ability to decompose a problem and specify its solution with precision.

---

Here's a couple of snippets from the [Tech Lead Journal #255 #255 - Stop Vibe Coding: Spec-Driven Development with The BMad Method - Brian Madison transcript](https://techleadjournal.dev/episodes/255/) that stood out to me:

> I said for this sprint, what we’re gonna do is everybody’s only gonna have one story. Each will be assigned a story. That’s not something we normally do in Agile. We want it to be more organic, but I said, in this case, everybody will have one story to do. This is a piece of work that would normally take you two to three days to do. But the only caveat is everybody has to use the agent mode. You cannot type any code. And if you get it done, I want you to do it over again and keep doing it and refining it.

> This was the most transformative sprint, because first of all, it gave everybody the permission to not have to worry about getting the thing done and moving on to the next thing. And even in a normal sprint, if you tell your developers and your teams, don’t worry, like learn how to use these tools, take some time to do it, a lot of developers will still self internalize that, yeah, well I can take the time. I really still need to get the work done. And that takes the priority. So by taking the pressure off for two weeks, it gave everybody this permission to fail. The permission to fail, right? The permission to take a chance and see what happens. Henry, when I tell you this was the most transformative thing I’ve ever seen, like it’s not an understatement. People started coming to the same conclusions. And basically inventing spec-driven development.

> Not only did people figure out how to get their stories and use agent modes, it was transformative in ways I did not even expect. All of a sudden people started saying, why do we need to do error triage manually every morning and look at the logs? We can now create a prompt that does that for us and pulls this information. Or, oh, there’s this thing called MCP. Why don’t we put an MCP in front of this tool? Before that, for months I was trying to get people to like, let’s be creative guys and just think of these things. Whatever it was about this exercise, not only did people start thinking about how to work with agent mode, but they started looking at all these opportunities to start automating their job away and it became like wildfire. And this is even before like there was a skills explosion or before the tools were good as they are now.

> So if I could give one piece of advice to every company, if you wanna transform your engineering team and get them to start using these tools, give them that space to do it. After that — this is no exaggeration — a few engineers went from never using the agent mode to using it 100% of the time, where they’ve taken it as their challenge to do everything with the agent mode. And this is not simple greenfield hobby projects. Like a lot of people think this only works with a greenfield or simple projects. This was people working in these legacy, you know, services that have existed for 5, 6, 7 years using the agent mode. Now since then, this has spread across the organization and it’s been transformative across product engineering and architecture and design. And, yeah, it’s just been magic. Like I can’t explain how useful that was.

> But what I really, you know, try to share with people, and I strongly believe this, and I think a lot of them are also kind of feeling this is, it’s just another abstraction, right? It’s like the switch from Assembly language to, I’ll just say to TypeScript, right? But this feels like almost that much of a gap. You’re able to step away a little bit. You’re almost becoming more of a leader of AI. You’re looking at it, you’re planning, you are thinking about a bigger picture. And I think it really frees you up to focus on more interesting challenges. You know, like how to do the for loop or how, you know, like what is the best way to structure this code. It’s still important, and as an experienced engineer it’s still important to keep those things in mind. But this also frees you up to think about it in a different way and plan.

> They really do appreciate that if you’ve never built software before, if you’ve never been an engineer, we take for granted as engineers, one of our greatest skills is the ability to take a problem and decompose it into small things, which is why we’re gonna work with AI so well. That is not something people take for granted.


