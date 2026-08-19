# Tool selection error

The agent picks the wrong tool for a task. The chosen tool either does not work, gives misleading output, or technically works but is inefficient.

## What it looks like

The agent needs to look up customer information. It has tools for `query_database` (structured) and `search_documents` (text search). It picks `search_documents` and gets back marketing material instead of the customer record.

Or: the agent has a fast `cached_lookup` and a slow `fresh_lookup`. It defaults to the slow one for every call. Functionally correct, but the system runs 10x slower than it should.

## Why it happens

- Tool descriptions are ambiguous, overlapping, or unclear
- The model defaults to the most "general-purpose"-sounding tool
- Tool names have unintended connotations (a tool called "advanced_search" sounds preferable to "simple_lookup", even when simple is correct)
- The model picks based on description tokens that happen to overlap with the query

## Detection signals

- Tool A is called when tool B would have been faster, cheaper, or more appropriate
- Tools that should be used roughly evenly are heavily skewed toward one
- Specific failure: the cited tool's output does not satisfy the user's question

Track tool selection distribution per task type. If `query_database` is called for 80% of lookups when only 30% should be database queries, you have a selection bias.

## Mitigations

### Clearer tool descriptions

Write tool descriptions that explicitly say when to use each tool and when not to:

```yaml
name: query_database
description: |
  Look up structured customer data: orders, billing, account status.
  USE THIS WHEN: the user asks about specific customer records,
    transaction history, or current account state.
  DO NOT USE: for searching documentation, FAQs, or marketing content.
    For those, use search_documents.
```

The negative space (do not use for X) matters more than the positive description.

### Disambiguating tool names

Rename tools to avoid overlap:
- `query_customer_database` instead of `query_database`
- `search_help_articles` instead of `search_documents`
- `cached_lookup_fast` instead of `cached_lookup`

The name should signal the use case, not just the mechanism.

### Tool decision examples in the system prompt

Add 2 to 3 worked examples of tool selection at the top of the system prompt:

```
Examples:
- "What is my last invoice?" -> query_customer_database
- "How do I cancel my subscription?" -> search_help_articles
- "Show me the rate limit policy" -> search_help_articles
```

This calibrates the model's default behavior.

### Tool composition over many specific tools

If the model frequently confuses 6 similar tools, consider collapsing them to 1 tool with a `type` parameter:

```yaml
name: lookup
parameters:
  type: enum [customer, billing, docs, faq]
  query: string
```

Fewer tool choices reduces selection error, at the cost of slightly more complex prompting.

### Reverse: split too-broad tools

If one tool is doing too much, the model picks it for everything. Split into specialized tools so the choice is more constrained.

The balance depends on the task; test both directions.

## When tool selection looks wrong but is right

Sometimes the model picks a "wrong" tool that works because the model is using it creatively:

- Using a search tool to discover available options before picking one
- Using a fast cached tool first, then a slow fresh tool only on cache miss
- Using a general tool because a specific one would error

These are not selection errors. They are reasonable strategies.

The distinction: in a real selection error, the tool's output does not answer the question. In creative usage, the model uses the result to inform its next choice.

## A note on tool-use models

Models trained heavily on tool use (Anthropic's tool-use mode, OpenAI function calling) make fewer selection errors than older models. If you are seeing high error rates with an older model, upgrading is often more impactful than improving descriptions.

The descriptions still need to be clear, but the model's baseline tool-use competence matters.

## A note on visibility

Many production agent systems log tool calls but not tool selection reasoning. If the model writes "I will use X because Y" before each tool call, that text tells you why the choice was made. Without it, all you see is "model called X" with no signal about whether the choice was deliberate.

Force tool selection rationale in the assistant turn before the tool call. Verbose but debuggable.
