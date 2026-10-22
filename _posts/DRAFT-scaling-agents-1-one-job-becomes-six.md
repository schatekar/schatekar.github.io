---
layout: post
title: "When one agent has to do six jobs"
excerpt: "An agent that does one job well will not stay that way once you ask it to do six. The instinct is to give it a longer prompt, a bigger model or more thinking time. Those help less than you hope. What works is breaking the job into smaller pieces, each handled by an agent that only sees what it needs."
tweet:
published: false
---

For the last two years, a large part of my work has been building agentic systems and then scaling them. Getting an agent to work in a demo is easy now. Getting it to do more, reliably, is where most of the effort goes. This series is about that second part.

The short version of this first post: an agent that does one job well will not stay that way once you ask it to do six. The instinct is to give it a longer prompt, a bigger model or more thinking time. Those help less than you hope. What works is breaking the job into smaller pieces, each handled by an agent that only sees what it needs.

The example I use in this series is not the system I built. It is close enough that every lesson carries over, and far enough that I'm not giving away anything that isn't mine to give. The numbers are made up too, though they are inspired by real ones.

## One agent, one job

Say you need to pull structured data out of legal contracts. You start small. One agent, one job: find the parties to the contract.

This works well. Your contracts are long, around 80 pages each, but the parties are named near the top. You get a prompt that does the job, you test it on a few hundred contracts and the results are good. Everyone is happy.

Then the scope grows. Now the business wants the parties, the obligations, the payment terms, the termination clauses, the liability caps and the renewal dates. Some of those are tricky. A clause can say that a new payment term takes effect next year, and the agent has to know the difference between what is true today and what will be true later.

One job has become six.

## What everyone tries first

Nobody's first thought is to break the agent down. It doesn't come naturally. You have an agent that works, so you try to make it do more. In my experience, people reach for these, roughly in this order:

1. Add more instructions to the prompt and keep tuning it.
2. Switch to a bigger model.
3. Turn up the reasoning effort. Most models now have a setting for how long they think before answering.
4. Run the whole thing as one long conversation, so the agent can work through the contract one section at a time.

Each of these is worth trying, and each has a catch.

A longer prompt means every instruction competes with every other one. The agent that was good at finding the parties now has five other jobs in its head while it does that one.

A bigger model and more reasoning cost more and take longer on every contract. If you are processing a few contracts, that is fine. If you are processing thousands, you feel it.

A long conversation has its own problems. As the context grows, the quality of the answers drops. This is often called context rot. And whatever the agent noted early in the conversation can go stale. It might hold on to a payment term from page 12 that page 60 changed.

None of these approaches fixes the real problem, which took us a while to see.

## Too much context, not too little capability

The contracts were too large to fit into a single agent's context. And even when one did fit, most of what the agent was reading had nothing to do with what it was looking for. The payment terms of an 80 page contract might live on three pages. The other 77 are noise for that job.

So we split the agent in two. A discovery agent finds the sections of the contract that are relevant. An extraction agent pulls the data out of those sections only. The extraction agent no longer sees 80 pages. It sees three.

Then we realised that even this was not enough. Finding the payment terms is a different job from finding the termination clauses. They are written differently and they sit in different places. So the split went one level further: a discovery agent and an extraction agent for each kind of data. Six kinds of data, six pairs of agents.

Each agent now has one job and only the context that job needs. The prompts got shorter. Each one could be tested on its own and changed without breaking the others.

## The problem you just created

Splitting one agent into twelve solves the context problem. It also gives you a new one. How do these agents hand work to each other? How do you know when a contract is finished? What happens when one of them fails halfway through?

That is the orchestration problem, and it gets its own post later in this series. Before that, the next post looks at a different kind of split, one that didn't need any of this pain to find.

If your agent works for one job and struggles with six, try giving it less to read before you give it more to think with.
