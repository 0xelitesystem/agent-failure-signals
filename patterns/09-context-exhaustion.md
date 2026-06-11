# Context window exhaustion

The agent's accumulated context grows large enough that the model's effective attention degrades, instructions get pushed out, or the request fails outright.

## What it looks like

Behaviors that emerge as context fills:

- Quality degrades around 60% to 70% of the window for most models, before any hard limit fires
- Earlier instructions become unreliable (see [instruction-drift](03-instruction-drift.md))
- Tool calls become less targeted
- The model starts repeating itself or losing track of what step it is on
- Eventually the model truncates the prompt or refuses to continue

## Why it happens

- Long agent runs accumulate every tool call, result, and assistant turn
- Tool results can be large (database queries, file reads, web pages)
- Without active context management, the conversation grows linearly forever
- Model "context window" is the hard limit; "effective attention" runs out earlier

## Detection signals

- Token count of prompt approaching the model's window limit
- Tool results that individually exceed a reasonable size (e.g., over 5K tokens for a single tool call)
- Quality metrics (task completion rate, instruction adherence) declining as task length increases
- Errors like `context_length_exceeded` or `prompt_too_long`

Track token count per step and per task. The trend tells you whether you have a runaway problem.

## Mitigations

### Tool result summarization

Long tool results should be summarized before going into the next prompt:

- A 10K-token database query result becomes a 500-token summary plus structured key data
- A 50K-token web page becomes a 2K-token summary of the relevant sections
- A directory listing of 500 files becomes "found 500 files matching X; top 10 by size are..."

The agent loses some information but the alternative is losing effective attention across the whole context.

### Conversation summarization

After every N turns, summarize the conversation so far into a compact "episodic memory":

```
Earlier in this conversation:
- User asked about customer X's order history
- Found 12 orders, summarized in [data store]
- Identified 2 disputes still pending
- Current task: draft response to dispute #487291

[continue from here]
```

Replace the verbose earlier history with this summary. The model still has the relevant context, just compressed.

### Per-tool result truncation

Limit how much each tool can return:

```python
def query_database(sql):
    results = db.execute(sql)
    if len(results) > 50:
        return {"results": results[:50], "total": len(results), "truncated": True}
    return {"results": results}
```

This forces the agent to request more specifically rather than dumping a large result into context.

### Selective context retention

Not everything needs to persist. After completing a sub-task, drop the intermediate tool calls and keep only the summary outcome.

For example, in a research agent: keep the final findings, drop the 15 individual web fetches that led to them.

### Smaller context windows on purpose

If your model supports it, sometimes a smaller context window forces better discipline. A 32K window with active summarization often produces better results than a 200K window used without management.

The bigger the window, the more tempting it is to skip context engineering. Discipline wins over capacity.

### Task decomposition

If a task systematically uses more than 50% of context, decompose into sub-tasks:

- Run sub-task 1 in a fresh context, persist the result
- Run sub-task 2 in a fresh context, given the result of sub-task 1 as input

Each sub-task runs with a clean window. The orchestration costs more in tokens overall but produces more reliable results.

## When long context is fine

Some tasks legitimately need the whole context:

- Document analysis where every part of the document matters
- Code refactoring where the whole file structure is relevant
- Investigations where any earlier finding might be referenced

For these, use a long-context-strong model (the model's behavior at high context utilization varies dramatically) and accept higher costs.

## A note on context cost

Long context is expensive even when it works. Input tokens cost real money; the 100th step of a 100-step agent run has carried all 99 prior steps as input.

Compute total token cost per task type. If a typical run uses 200K input tokens, your cost-per-task is dominated by context, not generation.

Reducing context length by 50% can halve your per-task cost. Sometimes a smaller, more disciplined system is more economical than a large unbounded one.

## A note on the "long context era"

Models with 200K+ context windows exist. They do not mean you should use them as a dumping ground.

Long-context capability is best used for "I genuinely need to attend to all of this", not "I do not want to think about context management". The latter degrades quality and costs more.
