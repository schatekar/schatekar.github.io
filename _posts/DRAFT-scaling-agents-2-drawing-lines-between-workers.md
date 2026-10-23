---
layout: post
title: "Splitting an agent before it becomes a god agent"
excerpt: "A research assistant started with a planner and one general worker. As the product grew, that worker picked up web search, document reading and API calls on top of its own reasoning. It was turning into a god agent. We split it by the kind of work, one worker per source of data, without waiting for it to struggle. Then we wrote down which agents may call which. Those rules made some bugs, like agents calling each other in a loop, impossible."
tweet:
published: false
---

This is the second post in a series on building and scaling agentic systems. In the first one, I stretched a single agent until it became a god agent and had to be split. This one is about a split we made before it got that far, and about the decision that comes straight after any split: which agents get to talk to which.

The short version: a research assistant started with a planner and one general worker. As the product grew, that worker picked up web search, document reading and API calls on top of its own reasoning. It was turning into a god agent. We split it by the kind of work, one worker per source of data, without waiting for it to struggle. Then we wrote down which agents may call which. Those rules made some bugs, like agents calling each other in a loop, impossible.

## The setup

I'll use a research assistant as the example in this post. It draws a close parallel to a production agentic system I have built.

A user asks it a long, detailed question. Something like "Find mid-sized logistics companies in Germany that opened a new warehouse in the last two years, check what the press said about them, compare them with the spreadsheet I've uploaded and tell me which ones look most exposed to rising fuel costs."

A question like that needs data from three places. Some comes from your own backend APIs, some from the open web and some from the file the user uploaded. Then something has to reason across all of it.

The first version had two kinds of agent. A planner broke the question into steps. A general worker took each step and did it. This is a common starting shape, and it worked.

## The worker kept growing

As the product evolved, more and more work landed on that one worker. It searched the web, scraped pages and made sense of what it found. It read through whatever the user uploaded. It called backend APIs, fetched data and reasoned about that data. And it still did the plain reasoning steps the planner handed it. It was on its way to becoming a god agent.

The prompt was the first place this showed. Every new responsibility meant more instructions, and every instruction had to sit next to all the others. But we didn't need the prompt to blow up before deciding to split it.

## You already know this rule

The single responsibility principle is one of the first design rules most of us learn as software engineers. A class should have one reason to change. It turns out an agent should too.

Searching the web and reasoning about the results is not the same job as looking through a user's documents and reasoning about those. Neither of them is the same as talking to a backend API. You can see that on a whiteboard before you write a single prompt. You don't need an eval to tell you.

So we drew lines.

## Where the lines went

1. The general worker kept the reasoning tasks the planner gave it, and nothing else.
2. The web worker searched the web, scraped pages and did a first pass of analysis on what it found.
3. The document worker dealt with the files the user uploaded.
4. The API worker called the backend APIs, fetched data and reasoned about it.
5. The memory worker stored data and fetched it back.

Each line follows a source of data, and that turned out to be a good place to cut. Every source has its own quirks. Web pages are messy and sometimes wrong. Uploaded documents come in every format a user can think of. APIs return structured data, sometimes far more of it than you asked for. A worker that deals with only one of these can have instructions about only that one.

## The second set of lines

Splitting the worker gave us five agents and a planner. That raised a question the single worker never had: which of them can call which?

It is tempting to let any agent call any other. A model will happily call whatever you give it. But every extra path is one more thing to test, and some paths make loops possible. One agent calls a second, the second calls a third, the third calls the first, and you pay for every turn until something times out.

So we wrote the rules down.

1. The planner hands steps to any of the workers.
2. The planner and the general worker can ask the memory worker what is already known.
3. The memory worker can call the web, document and API workers directly. It never calls the general worker or the planner.
4. The web, document and API workers can't call the memory worker.

## Memory that fetches what it is missing

Most of those rules exist because of the memory worker, so it is worth a closer look.

Other agents talk to it in plain language. They don't query a table. They ask a question, something like "what do we know about this company's fuel costs?" The memory worker checks whether what it already holds can answer that. If it can, it answers. If it can't, it doesn't hand the problem back to the caller. It calls the web, document or API worker itself to fill the gap, and then answers.

From the outside, that keeps things simple. The planner and the general worker don't need to know whether an answer came from memory or from a fresh fetch.

Two of the rules are what keep it safe.

1. The memory worker doesn't know who called it. It answers the question it was asked, the same way, whoever asked. That keeps its job narrow.
2. The source workers can't call the memory worker. The memory worker calls them, never the other way round. So a loop between memory and the sources is not unlikely, it is impossible. There is no path back.

There is more to memory than this: what it keeps, how it goes stale, and what happens when an agent needs to stop and wait for a person. That gets its own post later in the series.

## What it cost

Splitting did not make the system simpler. It moved the complexity somewhere else. With five workers, the planner has to decide which one to call, give each one the context it needs and put the results back together. And when one of them fails, something has to notice.

That is the orchestration problem. You don't get to skip it, and it is the subject of the next post.

If you can name the different jobs a worker is doing, you probably don't need to wait for it to struggle before you split it. And when you split it, write down who may call whom as well.
