---
layout: post
title: Saving tokens on repeated tasks in Claude Code
date: 2026-10-08 09:00:00 +0100
excerpt: "I recently had to run a task repeatedly for about 5000 items. The task involved fetching data from a database, finding anomalies, fixing them and then saving the data back to the database. The analysis was on slightly complex side and it was becoming clearer that it would benefit from a larger model like Opus 5.5."
tweet:
published: true
---

![Saving tokens on repeated tasks in Claude Code: one long conversation keeps growing, while a small main conversation with disposable agents stays flat](/images/claude-code-tokens/0-cover.png)

I recently had to run a task repeatedly for about 5000 items. The task involved fetching data from a database, finding anomalies, fixing them and then saving the data back to the database. The analysis was on slightly complex side and it was becoming clearer that it would benefit from a larger model like Opus 5.5.

I also needed summary stats from the task to be retained, which pushed me towards doing this in a single conversation. But doing this in a single conversation means a lot of context gets built up over time, and Claude Code sends the whole conversation with every request. The alternative was to clear and start a new conversation every now and then. That loses the running totals unless they are written to a file, and then you pay tokens again to read them back in.

![Keep the expensive work in a context you throw away: one long conversation grows every turn, while each agent run starts clean](/images/claude-code-tokens/1-context-growth.png)

The way I solved this was to build a Claude Code agent that knows how to perform the task, and pin it to Opus 5.5 at the effort level I wanted. The model and effort settings live in the agent's definition file, so you set them once and do not pass them on every call. Then I switched the model on the main conversation to Sonnet and invoked the agent in a loop.

Each run of the agent handles a small batch, 10 to 19 items at a time. It does all the heavy reading and fixing inside its own context and writes a one-line record per item to a file. I told it to end with a single line saying how many it finished. By default a subagent hands back its full result, so this part is something you have to ask for. When the run is over its context is gone, and the next run starts clean.

The main conversation never sees the raw data. It reads the status file, spot checks a couple of items from each batch, updates the running totals and starts the next batch. The stats live in files, so they survive whatever happens to the conversation.

![The loop, one batch at a time: the main conversation on Sonnet briefs an agent run on Opus 5.5, which writes status files that the main conversation reads and spot checks before the next batch](/images/claude-code-tokens/2-the-loop.png)

A few things I learned along the way.

The agent needs a very specific brief, including a hard stop on the batch size, because it will happily overshoot.

The spot checks were worth the tokens. They caught real mistakes the agent had made.

Check that the pin is respected. The agent's transcripts record the model and effort on every turn, and mine showed the pinned model on all of them.

It did not make the main conversation free. Mine still grew large, mostly because the spot checks and my own notes add up, and it ran on Opus for a while before I switched it. But the expensive part of the work, thousands of turns of reading and fixing, happened in contexts that were thrown away every few minutes instead of piling up. The agent turns averaged about 73k tokens of context. The main conversation averaged about 470k. My rough estimate is that the work processed three to seven times fewer tokens than it would have in one long conversation, before counting the price gap between the models. That is an estimate, not a measurement, and subagent tokens still count against your usage limits.

![Average context carried per turn: about 73k tokens for agent runs (about 6,700 turns) and about 470k for the main conversation (about 4,000 turns)](/images/claude-code-tokens/3-context-per-turn.png)

There is one more benefit, and I want to be honest about it. Pausing and resuming does not need agents. You can stop sending messages whenever you like, and you can resume a session later. But coming back to a big conversation after a break longer than the cache lifetime means the whole thing gets re-read, and that uses quota too. A small main conversation is cheap to come back to. Each batch is also a clean unit with its records in a file, so I can stop between batches without leaving anything half done, and pick the work up again later when my 5 hour or weekly window has room.

If you have a repeated task with a clear unit of work, try this before reaching for clear and restart.
