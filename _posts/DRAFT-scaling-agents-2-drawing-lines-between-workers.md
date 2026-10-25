---
layout: post
title: "Agents calling agents in one process"
excerpt: "Splitting a god agent doesn't always have to turn into a distributed system. In a research assistant I built, the agents called each other as tools, inside one process. That kept things simple. It also has limits, and one risk worth naming: recursion."
tweet:
published: false
---

This is the second post in a series on building and scaling agentic systems. In the first one, a god agent was split into twelve agents running on a queue, and that turned into a distributed system. This post looks at a split that stayed inside one process.

The short version: splitting a god agent doesn't always have to turn into a distributed system. In a research assistant I built, the agents called each other as tools, inside one process. That kept things simple. It also has limits, and one risk worth naming: recursion.

## Another god agent

I'll use a research assistant as the example in this post. It draws a close parallel to a production agentic system I have built.

A user asks it a long, detailed question. Something like "Find mid-sized logistics companies in Germany that opened a new warehouse in the last two years, check what the press said about them, compare them with the spreadsheet I've uploaded and tell me which ones look most exposed to rising fuel costs."

A question like that needs data from three places. Some comes from your own backend APIs, some from the open web and some from the file the user uploaded. Then something has to reason across all of it.

The first version had two kinds of agent. A planner broke the question into steps. A general worker took each step and did it.

As the product evolved, more and more work landed on that one worker. It searched the web, scraped pages and made sense of what it found. It read through whatever the user uploaded. It called backend APIs and reasoned about the data that came back. And it still did the plain reasoning steps the planner handed it.

That is a god agent again, but it got there by a different route. In the first post, the god agent came from too much context. This one came from too many responsibilities. That is even closer to the God Class it is named after. Nobody sets out to build a God Class. It grows one reasonable addition at a time.

The single responsibility principle says a class should have one reason to change. An agent should too. Searching the web and reasoning about the results is not the same job as reading a user's documents. Neither is the same as talking to a backend API. You can see that on a whiteboard before you write a single prompt.

## The split

So we split the general worker, one worker per source of data.

1. The general worker kept the reasoning tasks the planner gave it, and nothing else.
2. The web worker searched the web, scraped pages and did a first pass of analysis on what it found.
3. The document worker dealt with the files the user uploaded.
4. The API worker called the backend APIs, fetched data and reasoned about it.
5. The memory worker kept what the others had found and fetched it back when needed. It is interesting enough to get its own post.

Every source has its own quirks. Web pages are messy and sometimes wrong. Uploaded documents come in every format a user can think of. APIs return structured data, sometimes far more of it than you asked for. A worker that deals with only one of these can have instructions about only that one.

## Agents as tools

Here is where this system is different from the one in the first post. There is no queue. The planner calls each worker the same way it calls any other tool. The worker runs, returns its result and the planner carries on. All of it happens inside one process.

Most agent frameworks support two ways for one agent to involve another.

1. **Agent as a tool.** The caller asks another agent to do something, waits for the result and stays in charge. Some frameworks call this delegation, or a subagent call.
2. **Handoff.** Control moves to the other agent, which takes over from there. It usually gets the conversation history along with it.

We used the first. I built this with the OpenAI Agents SDK, but the same two patterns show up, under similar names, in LangGraph, Google's Agent Development Kit, Microsoft's Agent Framework, Pydantic AI, Strands and LlamaIndex. Some frameworks, like the Claude Agent SDK and CrewAI, only offer the first.

## Why one process

I wanted to start simple. It began as a proof of concept, and one process was the simplest thing that could work. It worked well enough that it carried into the product.

One process has a few things going for it. A worker's result goes straight back to the planner as an ordinary object. There is nothing to serialise, no queue between agents and no status table to tell you when the work is done. A whole request shows up as one trace, so when something goes wrong you can follow it from top to bottom.

Failures still happen, and you handle them in code. The planner retried a worker that failed. Depending on the kind of failure, the worker also had its own retry. Getting those two levels right is where most of the detail is.

What you give up is everything the queue in the first post gave us. If the process dies, the work in flight dies with it. There is no picking up from where it stopped. You can't scale one worker separately from the others. And deploying a new version is harder while requests are still running.

For a research assistant answering one question at a time, that trade was fine. For thousands of contracts running in the background, it wouldn't have been.

## Watch for recursion

When agents call each other as tools, the calls stack up the same way function calls do. That brings back an old problem.

The memory worker could call the web, document and API workers to fill gaps in what it knew. If the web worker could also call the memory worker, the two could keep calling each other. That is recursion without a base case.

Most frameworks give you limits for this: a maximum number of turns, a maximum depth or a budget. Those are worth setting, but they stop a loop after it has started. We went one step further. The web, document and API workers were never given the tool to call the memory worker. A loop between them was not unlikely. It was impossible, because the path didn't exist.

If you are giving agents other agents as tools, draw the call graph before you write the prompts. Any path that leads back to where it started needs either a base case or to not exist.

## What's next

The next post goes back to the queue, and to the problems the first post left open: handing over work, failures, retries and knowing when the work is done.
