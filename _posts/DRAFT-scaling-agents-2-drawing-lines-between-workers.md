---
layout: post
title: "Drawing lines between agents"
excerpt: "When one worker agent is doing several different kinds of work, split it by the kind of work. You don't have to wait for it to fail first. The split is the easy part. Getting the pieces to work together is where the effort goes."
tweet:
published: false
---

This is the second post in a series on building and scaling agentic systems. In the first one, I stretched a single agent until it had to be split. This one is about a split that never needed the stretching.

The short version: when one worker agent is doing several different kinds of work, split it by the kind of work. You don't have to wait for it to fail first. The split is the easy part. Getting the pieces to work together is where the effort goes.

The example in this post is not the system I built. It is close enough that the lessons carry over, and far enough that I'm not giving away anything that isn't mine to give.

## The setup

Picture a research assistant. A user asks it a long, detailed question. Something like "Find mid-sized logistics companies in Germany that opened a new warehouse in the last two years, check what the press said about them, compare them with the spreadsheet I've uploaded and tell me which ones look most exposed to rising fuel costs."

A question like that needs data from three places. Some comes from your own backend APIs, some from the open web and some from the file the user uploaded. Then something has to reason across all of it.

The first version had two kinds of agent. A planner broke the question into steps. A general worker took each step and did it. This is a common starting shape, and it worked.

## The worker kept growing

As the product evolved, more and more work landed on that one worker. It searched the web, scraped pages and made sense of what it found. It read through whatever the user uploaded. It called backend APIs, fetched data and reasoned about that data. And it still did the plain reasoning steps the planner handed it.

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
5. The memory worker stored data and fetched it back. There is more to this one than it sounds, and it gets its own post.

Each line follows a source of data, and that turned out to be a good place to cut. Every source has its own quirks. Web pages are messy and sometimes wrong. Uploaded documents come in every format a user can think of. APIs return structured data, sometimes far more of it than you asked for. A worker that deals with only one of these can have instructions about only that one.

## What it cost

Splitting did not make the system simpler. It moved the complexity somewhere else. With five workers, the planner has to decide which one to call, give each one the context it needs and put the results back together. And when one of them fails, something has to notice.

That is the orchestration problem. You don't get to skip it, and it is the subject of the next post.

If you can name the different jobs a worker is doing, you probably don't need to wait for it to struggle before you split it.
