---
layout:     post
title:      "AI Assisted coding"
description: "Are we using it properly"
date:    2026-04-20
author: "Paul"
publishDate: 2026-04-20
tags:
  - Programming
  - AI
  - Productivity
categories: [ code ]
draft: true
---

Since I began using AI to assist with coding, it didn't take long to realise that the more relevant information (context) available to the assistant the better the result. I've already started putting ADRs and design decisions etc in git alongside the code (goodbye Confluence!). 

This lead to the thoughts that the better I document the features I want then the better result I'll get - and once those features are properly documented, hopefully that makes maintenance easier too. This emphasises that spending more time on requirements - figuring out what it is that you want - leads to less time building/coding. And all of this would be true regardless of whether humans or AI are doing the implementing. 

Now though, I'm starting to realise how much I've been missing out on. While researching Spec Driven Development, I came across this podcast about BMAD. 

https://podcasts.apple.com/au/podcast/tech-lead-journal/id1523421550?i=1000762391704

I still think a lot of us aren’t using it properly. I know I haven’t. After listening to this podcast though, I am going to change things. Like all tools, we need to learn how to use them properly. These tools will get better over time, we’ve seen that happen SIGNIFICANTLY over the last 3 years.

This is only one such tool, there are many others - everyone is trying to solve this problem:

- https://github.com/addyosmani/agent-skills
- https://github.com/lilacmohr/ai-engineering-playbook
- https://github.com/gsd-build/get-shit-done
- https://github.com/obra/superpowers
- https://github.com/msitarzewski/agency-agents
- https://github.com/JohnCrickett/ai-assisted-engineering

I’m hoping to spend this afternoon building an app this way and see how it works and learn more about it. On the surface, it looks like a process we should follow anyway. While I'm impressed with results from just a series of simple prompts, I'm hoping a structured approach will provide better results - from an implementation and maintenance perspective.

I would however like to stay with the incremental approach instead of waterfall - to learn and verify as I go.

Also, another interesting podcast here about how DHH has changed his opinion about AI assisted coding from negative to positive. https://podcasts.apple.com/au/podcast/the-pragmatic-engineer/id1769051199?i=1000760299204

We aren’t there yet, but things are changing…


Here's a couple of snippets from the BMAD podcast transcript that stood out to me:

> I said for this sprint, what we’re gonna do is everybody’s only gonna have one story. Each will be assigned a story. That’s not something we normally do in Agile. We want it to be more organic, but I said, in this case, everybody will have one story to do. This is a piece of work that would normally take you two to three days to do. But the only caveat is everybody has to use the agent mode. You cannot type any code. And if you get it done, I want you to do it over again and keep doing it and refining it.

> This was the most transformative sprint, because first of all, it gave everybody the permission to not have to worry about getting the thing done and moving on to the next thing. And even in a normal sprint, if you tell your developers and your teams, don’t worry, like learn how to use these tools, take some time to do it, a lot of developers will still self internalize that, yeah, well I can take the time. I really still need to get the work done. And that takes the priority. So by taking the pressure off for two weeks, it gave everybody this permission to fail. The permission to fail, right? The permission to take a chance and see what happens. Henry, when I tell you this was the most transformative thing I’ve ever seen, like it’s not an understatement. People started coming to the same conclusions. And basically inventing spec-driven development.

> Not only did people figure out how to get their stories and use agent modes, it was transformative in ways I did not even expect. All of a sudden people started saying, why do we need to do error triage manually every morning and look at the logs? We can now create a prompt that does that for us and pulls this information. Or, oh, there’s this thing called MCP. Why don’t we put an MCP in front of this tool? Before that, for months I was trying to get people to like, let’s be creative guys and just think of these things. Whatever it was about this exercise, not only did people start thinking about how to work with agent mode, but they started looking at all these opportunities to start automating their job away and it became like wildfire. And this is even before like there was a skills explosion or before the tools were good as they are now.

> So if I could give one piece of advice to every company, if you wanna transform your engineering team and get them to start using these tools, give them that space to do it. After that — this is no exaggeration — a few engineers went from never using the agent mode to using it 100% of the time, where they’ve taken it as their challenge to do everything with the agent mode. And this is not simple greenfield hobby projects. Like a lot of people think this only works with a greenfield or simple projects. This was people working in these legacy, you know, services that have existed for 5, 6, 7 years using the agent mode. Now since then, this has spread across the organization and it’s been transformative across product engineering and architecture and design. And, yeah, it’s just been magic. Like I can’t explain how useful that was.

> But what I really, you know, try to share with people, and I strongly believe this, and I think a lot of them are also kind of feeling this is, it’s just another abstraction, right? It’s like the switch from Assembly language to, I’ll just say to TypeScript, right? But this feels like almost that much of a gap. You’re able to step away a little bit. You’re almost becoming more of a leader of AI. You’re looking at it, you’re planning, you are thinking about a bigger picture. And I think it really frees you up to focus on more interesting challenges. You know, like how to do the for loop or how, you know, like what is the best way to structure this code. It’s still important, and as an experienced engineer it’s still important to keep those things in mind. But this also frees you up to think about it in a different way and plan.

> They really do appreciate that if you’ve never built software before, if you’ve never been an engineer, we take for granted as engineers, one of our greatest skills is the ability to take a problem and decompose it into small things, which is why we’re gonna work with AI so well. That is not something people take for granted.


