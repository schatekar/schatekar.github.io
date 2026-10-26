---
layout: post
title: "Running agents on a queue"
excerpt: "Splitting one agent into twelve turned it into a distributed system. The problems that came with it, such as handing work over and retrying failed work, are the same ones any distributed system has. We solved them with a task queue, retries and a status table in a database."
tweet:
published: false
---

This is the third post in a series on building and scaling agentic systems. In the [first post](/when-one-agent-has-to-do-six-jobs/), a contract extraction agent grew until it had to be split into a discovery agent and an extraction agent for each kind of data. The [second post](/agents-calling-agents-in-one-process/) looked at a different split, where agents called each other as tools inside one process. This one goes back to the contracts, and to the problems that first split left open.

The short version: splitting one agent into twelve turned it into a distributed system. The problems that came with it, such as handing work over and retrying failed work, are the same ones any distributed system has. We solved them with a task queue, retries and a status table in a database.

## The problems from the first post

The first post ended with five problems that splitting the god agent created. Four of them are about orchestration, and they are what this post covers.

1. Handing work over from one agent to the next.
2. Failures.
3. Retries that repeat work.
4. Knowing when a contract has finished processing end to end.

The fifth, sharing context between agents, is a different kind of problem. I will cover it separately.

None of these four are new problems brought about by agentic AI. They are the same problems you get in any system where work is spread across many workers. That is good news, because the answers have been around for a long time.

## Handing work over

We built a queue-backed, event-driven system on Celery, a task queue for Python. Each agent ran as a Celery task, and work moved between agents as messages on a queue. A contract arriving put a task on the queue. The discovery agent for each kind of data picked it up, found the parts of the contract that mattered and put a task on the queue for the matching extraction agent.

No agent called another one directly. Each one did its job, said what should happen next and moved on. That is the opposite of the research assistant in the second post, where the planner called each worker and waited for the answer.

This setup also lets you scale each agent on its own. Celery can route each kind of task to its own queue, and each queue can have its own workers. If the payment terms extraction agent falls behind, you add workers for its queue and leave the rest alone. With every agent in one process, as in the [second post](/agents-calling-agents-in-one-process/), that wasn't possible.

Each agent was also packaged as its own Docker image, so it could be deployed separately from the others. A fix to one agent didn't mean redeploying the other eleven.

![A contract arrives and goes on a queue. Discovery agents for payment terms, termination and liability caps pick it up, six kinds in all. Each puts a task on its own queue for the matching extraction agent, and the extraction agents write to a results table](/images/scaling-agents/3-handoff.png)

## When an agent fails

Agents fail. A model call times out, a response comes back malformed or the agent simply doesn't finish the job. With one agent you would see this and run it again. With twelve running across thousands of contracts, you need the system to handle it.

Celery took care of most of it. If an agent failed or didn't finish, the task was retried a set number of times. If it still hadn't finished after the last retry, the whole job was marked as not completed. That gave someone a clear place to start: look at what went wrong, fix that agent and run the job again.

![Attempt 1 times out, retry 1 returns a malformed response, retry 2 doesn't finish, and the job is marked not completed](/images/scaling-agents/3-failures.png)

## Retries that repeat work

Retries raise a question of their own. If an agent wrote part of its output before it failed, does the retry write it again?

For us it didn't matter. Each agent wrote its output to a database table and overwrote whatever was there, so a retry replaced the earlier attempt instead of adding to it. Running an agent twice left the same result as running it once. That was a design decision that fit our requirements. It won't fit every system, but every system needs an answer to the question.

![Attempt 1 writes a partial payment terms result for a contract and then fails. The retry writes the same row, so the result is replaced, not added](/images/scaling-agents/3-retries.png)

## Knowing when a contract has finished processing

This one took more thought than I expected.

Every contract got a row in a database table that held the status of each agent working on it. When an agent finished, it wrote its status to that table. Then it checked whether every other agent was done too. If it was the last one, it put a message on the queue saying all agents were done. That message started the next stage of the workflow, the part that comes after the agents have done their work.

There is a catch. If two agents finish at almost the same moment, both can write their status, both can check and both can decide they were last. Or neither does. We used a database lock so that only one agent at a time could update the row and check it. Simple, and it worked.

![A status row with six kinds of data, four done and two running. Without a lock, both remaining agents write done, both check, and both send the all-done message. With a lock, the first agent sees the other still running and sends nothing, and the second sends exactly one message](/images/scaling-agents/3-finished.png)

## Ways to orchestrate agents

The queue is one way to do this. There are others, and each one suits a different kind of problem. I have used the first three. The fourth is what I call managed agentic workflow solutions, such as LangGraph, CrewAI Flows, Microsoft Agent Framework and Google's Agent Development Kit. I haven't used them myself, but they are common enough to belong in the comparison.

| | Inside an agent framework | On a task queue you run yourself | On a managed workflow service | On a managed agentic workflow solution |
|---|---|---|---|---|
| Example | OpenAI Agents SDK | Celery, plus a status table in a database | AWS Step Functions | LangGraph, CrewAI Flows, Microsoft Agent Framework, Google ADK |
| How work moves between agents | An agent calls other agents the same way it calls any tool | Each agent puts the next task on the queue | The workflow definition says which step runs next | The workflow is defined in code as steps and the links between them, and the steps share state |
| When an agent fails | Your code deals with it | Retries per task, then the job is marked as not completed | Retry rules are part of the workflow definition. When the last retry fails, the whole workflow is marked as not completed | Most let you set retries per step and save state as the run goes, so a failed run can resume where it stopped |
| Knowing when it is done | The calling agent decides it has its answer | You build it yourself: the status table, the lock and the last agent sending the message | The service keeps track for you | The run ends when the workflow reaches its last step |
| Fits best when | You want to start simple and the work fits inside one request | There is a lot of batch work and you want full control | The steps are known up front and you would rather not run the plumbing | The steps branch or loop, and you want the flow defined in code |

## What it cost

Splitting the agent fixed the context problem from the first post. In return, we had a distributed system to build and run.

1. **More to run.** A message broker, the Celery workers and the database behind the status table all have to be kept up and watched.
2. **Harder to follow one contract.** Its work is spread across twelve agents and their queues. Finding out why one result is wrong means piecing together logs from several places, not reading one call stack.
3. **Every agent has to be safe to run twice.** Retries only help if a second run doesn't spoil the result of the first.
4. **More to deploy.** Twelve images can be deployed separately, but each one also has to be built, versioned and kept compatible with the others.

That trade was worth it for us. A demo with one agent shows you none of this, so it is worth knowing about before you make the split.

If your agents are going to run as more than one piece, design them the way you would any other distributed system, and do it early.
