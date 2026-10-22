---
layout: post
title: "You split the agent. Now you have a distributed system."
excerpt: "Once one agent becomes many, you have a distributed system, whether you meant to build one or not. The agents are new. The plumbing is not, and most of what we already know about queues and retries still applies."
tweet:
published: false
---

This is the third post in a series on building and scaling agentic systems. In the first post, a single contract extraction agent grew until it had to be split into a discovery agent and an extraction agent for each kind of data. That solved the problem it was meant to solve. It also created a new one.

The short version: once one agent becomes many, you have a distributed system, whether you meant to build one or not. The agents are new. The plumbing is not, and most of what we already know about queues and retries still applies.

As before, the contract example is not the system I built. It runs closely parallel to it.

## Three questions you didn't have before

With one agent, you call it and wait. With a dozen, you have to answer three questions.

1. How do the agents hand work to each other?
2. What happens when one of them fails?
3. How do you know the whole job is finished?

None of these is an AI question. They are the same questions you would ask about any system where work is spread across many workers. That is good news, because the answers have been around for a long time.

## Each agent is a task on a queue

We packaged each agent as its own Docker image. At runtime, the agents ran as tasks in a queue-backed, event-driven system built on Celery. A contract arriving put a task on the queue. The discovery agent for each kind of data picked it up, found the parts of the contract that mattered, and put a task on the queue for the matching extraction agent.

No agent called another one directly. Each one did its job, said what should happen next and moved on.

## When an agent fails

Agents fail. A model call times out, a response comes back malformed, or the agent simply doesn't finish the job. With one agent you would see this and run it again. With a dozen running across many contracts, you need the system to handle it.

Celery took care of most of it. If an agent failed or didn't finish, the task was retried a set number of times. If it still hadn't finished after the last retry, the whole job was marked as not completed. That gave someone a clear place to start: look at what went wrong, fix that agent and run the job again.

Retries raise a question of their own. If an agent wrote part of its output before it failed, does the retry write it again? For us it didn't matter. Each agent wrote its output to a database table and overwrote whatever was there, so a retry replaced the earlier attempt instead of adding to it. That was a design decision that fit our requirements. It won't fit every system, but every system needs an answer to the question.

Nothing here is new. It is how you would treat any unreliable worker in a distributed system. The mistake would be to think agents need something more exotic.

## Knowing when the job is done

This one took more thought than I expected.

Every contract got a row in a database table that held the status of each agent working on it. When an agent finished, it wrote its status to that table. Then it checked whether every other agent was done too. If it was the last one, it put a message on the queue saying all agents were done. That message started the next stage of the workflow, the part that comes after the agents have done their work.

There is a catch. If two agents finish at almost the same moment, both can write their status, both can check, and both can decide they were last. Or neither does. We used a database lock so that only one agent at a time could update the row and check it. Simple, and it worked.

## Three ways I have orchestrated agents

The queue is one way to do this. I have used two others on different systems, and each one suits a different kind of problem.

| | Inside an agent framework | On a task queue you run yourself | On a managed workflow service |
|---|---|---|---|
| What I used | OpenAI Agents SDK | Celery, plus a status table in a database | AWS Step Functions |
| Where | The research assistant from the second post | The contract extraction system in this post | Another extraction pipeline |
| How work moves between agents | The planner either hands the task over to the next agent, or calls another agent the same way it calls any tool | Each agent puts the next task on the queue | The workflow definition says which step runs next |
| When an agent fails | Your code deals with it | Retries per task, then the job is marked as not completed | Retry rules are part of the workflow definition |
| Knowing when it is done | The planner decides it has its answer | You build it yourself: the status table, the lock and the last agent sending the message | The service keeps track for you |
| Fits best when | A user is waiting and the agents need to go back and forth | There is a lot of batch work and you want full control | The steps are known up front and you would rather not run the plumbing |
## What it cost

Splitting the agent fixed the context problem from the first post. In return, we had to build and run real infrastructure. That trade was worth it for us. It is worth knowing about before you make it, because a demo with one agent shows you none of this.

If your agents are going to run as more than one piece, design the plumbing the way you would for any other distributed system, and do it early.
