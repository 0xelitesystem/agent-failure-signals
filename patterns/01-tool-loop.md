# Tool-call loop

The agent calls the same tool with the same or near-identical arguments repeatedly, often increasing token count and cost with no progress toward the goal.

## What it looks like

The trace shows:
- N consecutive calls to the same tool
- Arguments that are identical, or differ only in trivial ways (a date format, a casing change, a comma)
- Each result is the same or substantially the same
- No clear plan adjustment in the assistant text between calls

A degenerate version: the agent keeps calling a tool that returns an error, then "thinking" about the error, then calling the same tool the same way again.

## Why it happens

- The tool result does not contain what the model expected, and the model assumes the call must have been wrong rather than the tool being inappropriate
- The model has no escape condition coded; the loop only breaks when context fills up or a turn limit fires
- A reflexive "I should verify by checking again" instinct from the prompt
- Tool documentation is unclear and the model is guessing argument formats

## Detection signals

Easy to detect mechanically:

- Same tool called 3+ times in a row with edit distance under N between arguments
- Same tool called 5+ times in a single agent run regardless of arguments
- Tool calls per token of progress (no progress markers, no final answer): track ratio

In your tracing system, surface these patterns automatically. A "loop suspected" tag costs nothing and catches the issue early.

## Mitigations

### Hard turn limit

Set a maximum tool calls per task. Common: 10 to 20. When hit, force the agent to either return a partial result or escalate to a human.

This is the single most reliable backstop. It does not fix the underlying issue but prevents unbounded cost.

### Repeat detection in the loop

After each tool call, check if the same tool was called with substantially the same arguments in the last N steps. If yes, inject an observation into the next prompt:

```
NOTE: You have called {tool} with similar arguments {N} times.
The tool is returning {summary of result}.
Either change your approach or conclude the task with the information you have.
```

This often unsticks the agent.

### Clearer tool descriptions

Many loops come from the model trying argument variations because the tool description was ambiguous. Look at the failing arguments; they tell you which part of the description is unclear.

### Result-aware prompting

If the tool returns a result indicating no match, no data found, or similar negative outcome, the system prompt should explicitly tell the model how to handle it. Without explicit guidance, the model often interprets a "no results" result as "the search query was wrong, try again".

### Tool error categorization

Distinguish tool errors that benefit from retry (rate limit, transient network) from errors that do not (bad arguments, no permission, no such resource). The agent should retry the first kind, escalate the second.

Tag your tool errors:
- `transient` - retry up to 3 times with backoff
- `permanent` - do not retry; report to user
- `argument` - do not retry; the model needs to change inputs based on the error

## When the loop is actually correct behavior

Sometimes "the same tool, similar arguments" is the right behavior:

- Pagination: same tool, different page numbers
- Polling: same status check until a state changes
- Search with refinement: same search tool with progressively narrowed queries

Build pagination, polling, and refinement into your tool design so they are not flagged as loops. A tool that takes a `next_page_token` is clearly paginating; a tool that returns 50 results and the agent calls it again with no change is looping.

## A note on cost

Loops are expensive. A single agent task that loops 30 times can cost more than 30 normal tasks combined because the context grows linearly while the work done is constant.

Budget alerts on per-task token usage catch loops faster than trace inspection in production.
