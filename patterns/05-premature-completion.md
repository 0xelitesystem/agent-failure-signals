# Premature completion

The agent declares the task complete and provides a final answer before all necessary work is done.

## What it looks like

User asks: "find the 3 customers with the highest churn risk and draft a retention email for each."

Agent runs the analysis, identifies 3 customers, drafts one email, and replies: "I have identified the top 3 at-risk customers and drafted a retention email for them."

The "for them" is doing a lot of work. There is one email, not three.

## Why it happens

- Tasks with implicit sub-tasks (drafting "emails" plural) require careful parsing
- The model treats sub-task 1 as representative of the whole and stops
- "Good enough" instinct trained on examples where users were satisfied with partial work
- Long tasks invite shortcuts in the final stretch

## Detection signals

Hard to detect generically; depends on task structure. Some heuristics:

- User request includes a number ("3 emails", "5 issues") and agent output count does not match
- User request has multiple verbs ("analyze and summarize and send") and one verb is absent from the trace
- Final message uses generalizing language ("for them", "for each", "across the dataset") without listing individually

A useful technique: parse the user request for explicit counts and required deliverables, then verify the trace produces them.

## Mitigations

### Explicit deliverable checklist

Before starting the task, force the agent to enumerate deliverables:

```
Before you start, list everything you need to produce as a numbered checklist.
Check off each item as you complete it. Do not return a final answer until
every item is checked off.
```

This often catches the issue. The agent that says "I will produce: (1) ranked list of 3 customers, (2) email for customer 1, (3) email for customer 2, (4) email for customer 3" rarely forgets to do all 4.

### Structured output requirement

Define the final output schema. If the schema requires an array of 3 emails, the agent cannot return one and claim it is done.

### Validation step

After the agent declares completion, run a deterministic check: does the output match the expected structure? Does it have the requested count? If not, send back a "the output is incomplete: X is missing" message and let the agent continue.

This adds a turn but catches premature completion reliably.

### Avoid ambiguous user requests

When a user request is ambiguous about scope, the agent should ask first, not assume. "Did you want one email or three?" prevents the failure mode at the source.

System prompt addition: "If the user request is ambiguous about how many of something to produce, ask before starting. Do not assume."

### Re-read the original request before finalizing

A simple last-step prompt: "before declaring the task complete, re-read the user's original request. Have you addressed every part? List what you produced; list what they asked for; confirm match."

This adds one turn and catches many premature completions.

## When partial completion is right

Sometimes the agent is supposed to return partial work:

- The user said "start with the first one and let me check before continuing"
- The task is too large for one run and the agent has communicated this clearly
- Some sub-tasks failed and were correctly skipped (with explanation)

The distinction: legitimate partial completion is acknowledged. The agent says "I completed 1 of 3; the other 2 require X which I cannot do". Premature completion claims the whole task is done.

## A note on chain prompts

Multi-step pipeline prompts (where step 1 produces input for step 2, step 2 for step 3) suffer from premature completion at the boundary between steps. The output of step 1 is often "partial" in step 2's view but "complete" in step 1's.

Solve this with explicit hand-off contracts: step 1 produces output in a schema that step 2 can validate as complete. If the schema is missing required fields, step 2 rejects the input and step 1 redoes its work.
