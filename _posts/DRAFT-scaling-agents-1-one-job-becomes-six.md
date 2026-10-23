---
layout: post
title: "When one agent has to do six jobs"
excerpt: "An agent that does one job well will not stay that way once you ask it to do six. The instinct is to give it a longer prompt, a bigger model or more thinking time. Those help, up to a point. Past that point, they stop paying off. What works is breaking the job into smaller pieces, each handled by an agent that only sees what it needs."
tweet:
published: false
---

For the last two years, a large part of my work has been building agentic systems and then scaling them. Getting an agent to work in a demo is easy now. Getting it to do more, reliably, is where most of the effort goes. This series is about that second part.

The short version of this first post: an agent that does one job well will not stay that way once you ask it to do six. The instinct is to give it a longer prompt, a bigger model or more thinking time. Those help, up to a point. Past that point, they stop paying off. What works is breaking the job into smaller pieces, each handled by an agent that is very good at that one thing. That brings its own challenges. Some are new. Others can be handled with what we have learned about distributed system design over the years.

## One agent, one job

I'll use data extraction from legal contracts as the example in this post. It draws a close parallel to a production agentic system I have built.

We started small. One agent, one job: find the parties to the contract.

It worked well. The contracts were long, often 80 pages or more, but the parties are named near the top. We got a prompt that did the job, tested it on a few hundred contracts and the results were good.

Then the scope grew. The business wanted the parties, the obligations, the payment terms, the termination clauses, the liability caps and the renewal dates. Some of those are tricky. A clause can say that a new payment term takes effect next year, and the agent has to know the difference between what is true today and what will be true later.

One job had become six.

## What we tried first

Breaking the agent down was not our first thought. I don't think it is anyone's. You have an agent that works, so you try to make it do more. We went through the obvious options, roughly in this order:

1. Add more instructions to the prompt and keep tuning it.
2. Switch to a bigger model.
3. Turn up the reasoning effort. Most models now have a setting for how long they think before answering.

> **A sidenote.** I call this the "God Agent Anti-Pattern", on the lines of the famous "God Class" or "God Object" anti-pattern we have known for a long time. You will be amazed how strikingly similar both are to each other.

Each of these helped a little, and each had a catch.

A longer prompt means every instruction competes with every other one. The agent that was good at finding the parties is now a god agent, with five other jobs in its head while it does that one.

A bigger model and more reasoning cost more and take longer on every contract. If you are processing a few contracts, that is fine. If you are processing thousands, you feel it.

We also considered a fourth option, though we didn't go far with it: run the whole thing as one long conversation. It is natural to picture a person chatting with a bot when you hear that, but it doesn't have to be. A Python loop can send the agent a fixed set of messages, one after another. Find the parties. Now find the payment terms. Now the termination clauses. Every answer stays in the context for the next question. It is still a god agent. It just does its jobs one turn at a time.

That has its own problems. As the context grows, the quality of the answers drops. This is often called context rot. Whatever the agent noted early in the conversation can also go stale. It might hold on to a payment term from page 12 that page 60 changed. And sooner or later you hit the model's context window limit. At that point you can start over or compact the context. Starting over loses what the agent has worked out so far. Compacting means a summary decides what the agent remembers. Neither is great.

None of these fixed the real problem.

## Too much context, not too little capability

Some contracts were too long to fit into the god agent's context. And even when one did fit, most of what the agent was reading had nothing to do with what it was looking for. The payment terms of an 80 page contract might live on three pages. The other 77 are noise for that job.

> **A sidenote.** Even those three pages are rarely enough on their own. The payment terms only make full sense alongside facts from other parts of the contract: who the parties are, which jurisdiction applies, how the terms are defined. Once the work is split across agents, that information sits with a different agent. Splitting cuts out the noise. It also splits the context. So how do agents share what they know without each one going back to read all 80 pages?

So we split the god agent in two. A discovery agent finds the sections of the contract that are relevant. An extraction agent pulls the data out of those sections only. The extraction agent no longer sees 80 pages. It sees far less.

Even with the split, we could not reach our quality threshold across a broad range of contracts. Finding the payment terms is a different job from finding the termination clauses. They are written differently and they sit in different places. So the split went one level further: a discovery agent and an extraction agent for each kind of data. Six kinds of data, six pairs of agents.

Each agent now has one job and only the context that job needs. The prompts got shorter. Each one could be tested on its own and changed without breaking the others.

## The problem we just created

Splitting the god agent into twelve solves the context problem. It also gives us new ones. Most of them have nothing to do with AI. They are the problems you get in any system where work is spread across many workers.

1. **Handing work over.** The god agent used to do everything in one go. Now twelve agents have to pass work to each other in the right order. Each extraction agent waits on its discovery agent, and something has to carry the result from one to the other.
2. **Failures.** A model call times out, or an agent returns something malformed. With one agent you would just run it again. With twelve agents working through thousands of contracts, the system has to notice, retry on its own and know when to give up.
3. **Retries that repeat work.** If an agent wrote part of its output before it failed, the retry can write it again. You need to decide what happens to the first attempt. In other words, each agent's work has to be idempotent: running it twice should leave the same result as running it once.
4. **Knowing when a contract is finished.** Twelve agents finish at different times. Something has to know when the last one is done so the next stage can start. If two finish at the same moment, both can think they were last. Or neither does.
5. **Sharing context.** This is the sidenote from earlier. Each agent now sees a small part of the contract, but some of what it needs was found by another agent.

The first four are the orchestration problem. How I solved them gets its own post later in this series. Sharing context across multiple agents is an interesting problem of its own. I will talk about my approach to it in a separate post. Before that, the next post looks at a different kind of split, one that didn't need any of this pain to find.

If your agent works for one job and struggles with six, try giving it less to read before you give it more to think with.
