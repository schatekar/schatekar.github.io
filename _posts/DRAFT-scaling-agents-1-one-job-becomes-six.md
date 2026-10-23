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

It worked well. The contracts were long, around 80 pages each, but the parties are named near the top. We got a prompt that did the job, tested it on a few hundred contracts and the results were good.

Then the scope grew. The business wanted the parties, the obligations, the payment terms, the termination clauses, the liability caps and the renewal dates. Some of those are tricky. A clause can say that a new payment term takes effect next year, and the agent has to know the difference between what is true today and what will be true later.

One job had become six.

## What we tried first

Breaking the agent down was not our first thought. I don't think it is anyone's. You have an agent that works, so you try to make it do more. We went through the obvious options, roughly in this order:

1. Add more instructions to the prompt and keep tuning it.
2. Switch to a bigger model.
3. Turn up the reasoning effort. Most models now have a setting for how long they think before answering.

Each of these helped a little, and each had a catch.

A longer prompt means every instruction competes with every other one. The agent that was good at finding the parties now has five other jobs in its head while it does that one.

A bigger model and more reasoning cost more and take longer on every contract. If you are processing a few contracts, that is fine. If you are processing thousands, you feel it.

We also considered a fourth option, though we didn't go far with it: run the whole thing as one long conversation. It is natural to picture a person chatting with a bot when you hear that, but it doesn't have to be. A Python loop can send the agent a fixed set of messages, one after another. Find the parties. Now find the payment terms. Now the termination clauses. Every answer stays in the context for the next question.

That has its own problems. As the context grows, the quality of the answers drops. This is often called context rot. Whatever the agent noted early in the conversation can also go stale. It might hold on to a payment term from page 12 that page 60 changed. And sooner or later you hit the model's context window limit. At that point you can start over or compact the context. Starting over loses what the agent has worked out so far. Compacting means a summary decides what the agent remembers. Neither is great.

None of these fixed the real problem.

## Too much context, not too little capability

The contracts were too large to fit into a single agent's context. And even when one did fit, most of what the agent was reading had nothing to do with what it was looking for. The payment terms of an 80 page contract might live on three pages. The other 77 are noise for that job.

So we split the agent in two. A discovery agent finds the sections of the contract that are relevant. An extraction agent pulls the data out of those sections only. The extraction agent no longer sees 80 pages. It sees three.

Then we realised that even this was not enough. Finding the payment terms is a different job from finding the termination clauses. They are written differently and they sit in different places. So the split went one level further: a discovery agent and an extraction agent for each kind of data. Six kinds of data, six pairs of agents.

Each agent now has one job and only the context that job needs. The prompts got shorter. Each one could be tested on its own and changed without breaking the others.

## The problem you just created

Splitting one agent into twelve solves the context problem. It also gives you a new one. How do these agents hand work to each other? How do you know when a contract is finished? What happens when one of them fails halfway through?

That is the orchestration problem, and it gets its own post later in this series. Before that, the next post looks at a different kind of split, one that didn't need any of this pain to find.

If your agent works for one job and struggles with six, try giving it less to read before you give it more to think with.
