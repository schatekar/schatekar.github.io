---
layout: post
title: "Running agents on a queue"
excerpt: "Once one agent becomes many, you have a distributed system, whether you meant to build one or not. The agents are new. The plumbing is not, and most of what we already know about queues and retries still applies."
tweet:
published: false
---

This is the third post in a series on building and scaling agentic systems. In the [first post](/when-one-agent-has-to-do-six-jobs/), a contract extraction agent grew until it had to be split into a discovery agent and an extraction agent for each kind of data. The [second post](/agents-calling-agents-in-one-process/) looked at a different split, where agents called each other as tools inside one process. This one goes back to the contracts, and to the problems that first split left open.

The short version: once one agent becomes many, you have a distributed system, whether you meant to build one or not. The agents are new. The plumbing is not, and most of what we already know about queues and retries still applies.

## The problems from the first post

The first post ended with five problems that splitting the god agent created. Four of them are about orchestration, and they are what this post covers.

1. Handing work over from one agent to the next.
2. Failures.
3. Retries that repeat work.
4. Knowing when a contract is finished.

The fifth, sharing context between agents, is a different kind of problem. I will cover it separately.

None of these four is an AI problem. They are the same problems you get in any system where work is spread across many workers. That is good news, because the answers have been around for a long time.

## Handing work over

We packaged each agent as its own Docker image. At runtime, the agents ran as tasks in a queue-backed, event-driven system built on Celery. A contract arriving put a task on the queue. The discovery agent for each kind of data picked it up, found the parts of the contract that mattered and put a task on the queue for the matching extraction agent.

No agent called another one directly. Each one did its job, said what should happen next and moved on. That is the opposite of the research assistant in the second post, where the planner called each worker and waited for the answer.

![A contract arrives and goes on a queue. Discovery agents for payment terms, termination and liability caps pick it up, six kinds in all. Each puts a task on its own queue for the matching extraction agent, and the extraction agents write to a results table](/images/scaling-agents/3-handoff.png)

## When an agent fails

Agents fail. A model call times out, a response comes back malformed or the agent simply doesn't finish the job. With one agent you would see this and run it again. With twelve running across thousands of contracts, you need the system to handle it.

Celery took care of most of it. If an agent failed or didn't finish, the task was retried a set number of times. If it still hadn't finished after the last retry, the whole job was marked as not completed. That gave someone a clear place to start: look at what went wrong, fix that agent and run the job again.

## Retries that repeat work

Retries raise a question of their own. If an agent wrote part of its output before it failed, does the retry write it again?

For us it didn't matter. Each agent wrote its output to a database table and overwrote whatever was there, so a retry replaced the earlier attempt instead of adding to it. Running an agent twice left the same result as running it once. That was a design decision that fit our requirements. It won't fit every system, but every system needs an answer to the question.

![Top: attempt 1 times out, retry 1 returns a malformed response, retry 2 doesn't finish, and the job is marked not completed. Bottom: attempt 1 writes a partial payment terms result for a contract and fails. The retry writes the same row, so the result is replaced, not added](/images/scaling-agents/3-failures-retries.png)

## Knowing when a contract is finished

This one took more thought than I expected.

Every contract got a row in a database table that held the status of each agent working on it. When an agent finished, it wrote its status to that table. Then it checked whether every other agent was done too. If it was the last one, it put a message on the queue saying all agents were done. That message started the next stage of the workflow, the part that comes after the agents have done their work.

There is a catch. If two agents finish at almost the same moment, both can write their status, both can check and both can decide they were last. Or neither does. We used a database lock so that only one agent at a time could update the row and check it. Simple, and it worked.

![A status row with six kinds of data, four done and two running. Without a lock, both remaining agents write done, both check, and both send the all-done message. With a lock, the first agent sees the other still running and sends nothing, and the second sends exactly one message](/images/scaling-agents/3-finished.png)

## Three ways I have orchestrated agents

The queue is one way to do this. I have used two others, and each one suits a different kind of problem.

| | Inside an agent framework | On a task queue you run yourself | On a managed workflow service |
|---|---|---|---|
| What I used | OpenAI Agents SDK | Celery, plus a status table in a database | AWS Step Functions |
| Where | The research assistant from the second post | The contract extraction system in this post | Another extraction pipeline |
| How work moves between agents | The planner calls other agents the same way it calls any tool | Each agent puts the next task on the queue | The workflow definition says which step runs next |
| When an agent fails | Your code deals with it | Retries per task, then the job is marked as not completed | Retry rules are part of the workflow definition |
| Knowing when it is done | The planner decides it has its answer | You build it yourself: the status table, the lock and the last agent sending the message | The service keeps track for you |
| Fits best when | You want to start simple and the work fits inside one request | There is a lot of batch work and you want full control | The steps are known up front and you would rather not run the plumbing |

## What it cost

Splitting the agent fixed the context problem from the first post. In return, we had to build and run real infrastructure. That trade was worth it for us. It is worth knowing about before you make it, because a demo with one agent shows you none of this.

If your agents are going to run as more than one piece, design the plumbing the way you would for any other distributed system, and do it early.
