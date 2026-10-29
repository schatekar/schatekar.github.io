---
layout: post
title: "Agents calling agents in one process"
date: 2026-10-25 21:42:00 +0000
image: /images/scaling-agents/2-cover.png
excerpt: "Splitting a god agent doesn't always have to turn into a distributed system. In a research assistant I built, the agents called each other as tools, inside one process. That kept things simple. It also has limits, and one risk worth naming: recursion."
tweet:
published: true
---

This is the second post in a series on building and scaling agentic systems. In the [first one](/when-one-agent-has-to-do-six-jobs/), a god agent was split into twelve agents running on a queue, and that turned into a distributed system. This post looks at a split that stayed inside one process.

The short version: splitting a god agent doesn't always have to turn into a distributed system. In a research assistant I built, the agents called each other as tools, inside one process. That kept things simple. It also has limits, and one risk worth naming: recursion.

## Another god agent

I'll use a research assistant as the example in this post. It draws a close parallel to a production agentic system I have built.

It started with a narrow scope. A user asked a question about data we already had, and the assistant fetched that data from our backend APIs and reasoned about it. There were two kinds of agent. A planner broke the question into steps. A general worker took each step and did it, calling the APIs when a step needed data.

That worked, so the scope grew. Users wanted answers that went beyond our own data, so the worker learned to search the web, scrape pages and make sense of what it found. Then users wanted to bring their own data, so it learned to read through whatever they uploaded. Each addition made sense on its own. Each one went to the worker, because that was the agent that did the work.

Before long, users were asking questions like "Find mid-sized logistics companies in Germany that opened a new warehouse in the last two years, check what the press said about them, compare them with the spreadsheet I've uploaded and tell me which ones look most exposed to rising fuel costs." A question like that needs data from all three places, and then something has to reason across all of it. The same worker was doing every part of it, along with the plain reasoning steps the planner handed it.

![The planner calls one general worker as a tool. That worker talks to the backend APIs, the web and uploaded files, and does the reasoning steps itself](/images/scaling-agents/2-one-worker.png)

That is a god agent again, but it got there by a different route. In the first post, the god agent came from too much context. This one came from too many responsibilities. That is even closer to the God Class it is named after. Nobody sets out to build a God Class. It grows one reasonable addition at a time.

The single responsibility principle says a class should have one reason to change. An agent should too. Searching the web and reasoning about the results is not the same job as reading a user's documents. Neither is the same as talking to a backend API. That is easy to see on a whiteboard. It is much harder to see while you are adding one feature at a time.

## The split

So we split the general worker. Each source of data got its own worker, and a memory worker sat in front of them.

1. The general worker kept the reasoning tasks the planner gave it, and nothing else.
2. The memory worker was where the other agents went for data. It checked what was already known first. If something was missing, it asked the right source worker. It is interesting enough to get its own post.
3. The web worker searched the web, scraped pages and did a first pass of analysis on what it found.
4. The document worker dealt with the files the user uploaded. [Layout-aware chunking](/layout-aware-chunking-and-retrieval/) helps a worker like this find the right part of a long file.
5. The API worker called the backend APIs, fetched data and reasoned about it.

Every source has its own quirks. Web pages are messy and sometimes wrong. Uploaded documents come in every format a user can think of. APIs return structured data, sometimes far more of it than you asked for. A worker that deals with only one of these can have instructions about only that one.

![The planner calls the general worker and the memory worker. The general worker calls the memory worker. The memory worker calls the web, document and API workers](/images/scaling-agents/2-split-workers.png)

## Agents as tools

Here is where this system is different from the one in the first post. There is no queue. The planner calls the general worker and the memory worker the same way it calls any other tool. The general worker calls the memory worker the same way, and so does the memory worker when it needs a source worker. Each worker runs, returns its result and the caller carries on. All of it happens inside one process.

Most agent frameworks support two ways for one agent to involve another.

1. **Agent as a tool.** The caller asks another agent to do something, waits for the result and stays in charge. Some frameworks call this delegation, or a subagent call.
2. **Handoff.** Control moves to the other agent, which takes over from there. It usually gets the conversation history along with it.

We used the first. I built this with the OpenAI Agents SDK, but the same two patterns show up, under similar names, in LangGraph, Google's Agent Development Kit, Microsoft's Agent Framework, Pydantic AI, Strands and LlamaIndex. Some frameworks, like the Claude Agent SDK and CrewAI, only offer the first.

## Why one process

I wanted to start simple. It began as a proof of concept, and one process was the simplest thing that could work. It worked well enough that it carried into the product.

Failures still happen, and you handle them in code. The planner retried a worker that failed. Depending on the kind of failure, the worker also had its own retry. Getting those two levels right is where most of the detail is.

## Watch for recursion

When agents call each other as tools, the calls stack up the same way function calls do. That brings back an old problem.

The memory worker could call the web, document and API workers to fill gaps in what it knew. If the web worker could also call the memory worker, the two could keep calling each other. That is recursion without a base case.

Most frameworks give you limits for this: a maximum number of turns, a maximum depth or a budget. Those are worth setting, but they stop a loop after it has started. We went one step further. The web, document and API workers were never given the tool to call the memory worker. A loop between them was not unlikely. It was impossible, because the path didn't exist.

If you are giving agents other agents as tools, draw the call graph before you write the prompts. Any path that leads back to where it started needs either a base case or to not exist.

## What one process gives you

Starting simple was my reason, but it is not the only good one. Keeping a multi-agent workflow in one process has real advantages.

1. **It is the simplest thing that works.** There is no queue, no separate deployment for each worker and no status table. You can add those later if you need them.
2. **Passing data is easy.** A worker's result goes straight back to the caller as an ordinary object. There is nothing to serialise and nothing to store in between.
3. **It is fast.** There is no queue hop between agents. That adds up when the planner calls workers many times to answer one question.
4. **One trace per request.** The whole request shows up as one tree of calls. When something goes wrong, you can follow it from top to bottom.
5. **Failures are handled in code.** A retry is a few lines next to the call it protects. You decide what to retry and how many times. That is where our two levels of retry lived.

## What it costs

Most of what you give up is what the queue in the first post gave us.

1. **A crash loses the work in flight, unless you plan for it.** If the process dies, every agent call in it dies too. Picking up from where it stopped is possible. Anthropic [describes doing that](https://www.anthropic.com/engineering/multi-agent-research-system) in its own multi-agent research system, with regular checkpoints so agents resume instead of starting over. But you have to build it yourself. A queue gives you much of it for free.
2. **Every agent ships together.** The agents share a process, so they share a deployment. Deploying safely is still possible, with blue/green for example, if the workflow is packaged and hosted for it. But you can't roll out or roll back one worker on its own. A change to the web worker means redeploying all of them.
3. **You can't scale one worker on its own.** Every worker runs as many instances as the process does. If you need ten general workers running at once and only two of each of the others, you can't have that. You run ten copies of everything.
4. **One bad worker affects the rest.** All the agents share the same process. A worker that hangs or eats memory slows down or takes down the others with it.

For a research assistant answering one question at a time, those costs were fine. For the legal contract extraction in the [first post](/when-one-agent-has-to-do-six-jobs/), with thousands of contracts running in the background, they wouldn't have been.

## What's next

The [next post](/running-agents-on-a-queue/) goes back to the queue. It covers how I solved the problems the first post left open: handing over work, failures, retries and knowing when the work is done. With one process, you can avoid most of these for a while. With a queue, you have to solve them from day one.
