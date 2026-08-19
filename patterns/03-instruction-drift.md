# Instruction drift

The agent follows its system instructions early in the task. After several tool calls or a long stretch of context, those instructions are no longer followed reliably.

## What it looks like

The system prompt says, for example, "always cite chunk IDs", "never offer financial advice", "respond in JSON only". For the first 3 to 5 turns, the agent does this. By turn 10 or 15, citations stop appearing, advice creeps in, JSON formatting breaks.

The agent is not ignoring instructions intentionally. The instructions have effectively faded from attention.

## Why it happens

- Long contexts dilute instruction salience
- Recent tool results loom larger in attention than the system prompt
- Model architectures vary in how strictly they attend to early-context content
- Conversational momentum pulls the agent toward natural language even when JSON was specified

## Detection signals

For each instruction in the system prompt, define a programmatic check:

- "respond in JSON" -> parse each response as JSON; flag failures
- "cite every claim" -> count claims and citations per response; ratio should stay high
- "no financial advice" -> keyword search for advice patterns
- "respond in under 200 words" -> word count

Run these on every step. The drift point in the trace is usually visible: instructions held for steps 1 to 7, broke at step 8.

## Mitigations

### Repeat critical instructions near the bottom of the prompt

Move the most important instructions to just before the assistant turn. The model attends most strongly to the end of the context. A "REMINDER: respond in JSON only" right before generation is more effective than the same text at the top of a long system prompt.

### Per-turn instruction injection

Instead of relying on the system prompt to persist, append a short reminder to the user message or as a tool result preamble:

```
[Reminder: respond in JSON only. Cite every claim with chunk IDs.]
```

Cheap, effective for the highest-priority rules.

### Shorter context

Drift correlates with context length. If a long context is causing drift, the fix is often to summarize older turns into a compact form (an episodic memory) rather than carrying the full history forward.

### Structured outputs

If the instruction is "respond in JSON", use the model's structured output mode (where available) instead of prompting. Structured output mode does not drift; it is enforced at decoding time.

### Test for drift, not just compliance

Most teams test instruction-following on short prompts and assume long-context behavior is the same. Build long-context test cases that include 20 to 30 turns of conversation history before the final test message. Drift only shows up under load.

### Different model

Some models hold instructions better than others. If your eval shows drift at turn N for model A but turn 2N for model B, the cost difference may be worth it for tasks that need stability.

## A note on contradicting instructions

Sometimes drift is not drift; the model is correctly handling a contradiction.

Example: system prompt says "always be concise". User says "explain in detail". The model elaborates. This is not drift; it is the model correctly prioritizing the user's request over a default.

Distinguish drift from contradiction handling. Build instructions with explicit priority levels:

```
RULES (in order of priority):
1. Never provide medical diagnosis (NON-NEGOTIABLE).
2. Respond in JSON. If the user asks for prose, ignore this rule.
3. Cite every claim. If citations would prevent answering, omit them.
```

## A note on agentic loops

In multi-step agent loops, the system prompt is reread on every step. Instruction drift in this setting is more often "instructions got pushed out by tool results" than "the model forgot".

Inspect the actual prompt sent on the drift turn. If the system prompt is intact, the model is choosing not to follow it. If the system prompt is truncated or pushed past the model's effective attention window, the fix is context management, not prompting.
