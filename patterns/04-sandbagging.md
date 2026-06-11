# Sandbagging

The agent claims a task is complete or impossible, when in fact it stopped at the first significant friction and did not attempt available alternatives.

## What it looks like

User asks the agent to do X. The agent tries one approach, hits an obstacle, and replies: "I was unable to complete X because Y. Here are some things you could try yourself."

Investigation shows: the obstacle was bypassable with one of 3 available tools the agent did not use. Or a different query. Or asking the user a clarifying question.

The agent did not lie. It just gave up faster than the user expected.

## Why it happens

- The model's training included a lot of "if uncertain, defer to the user" examples
- Tool errors trigger conservative behavior
- Helpfulness training and harmlessness training pull in different directions; harmlessness wins when the agent senses risk
- The model lacks confidence that the user actually wants more attempts

## Detection signals

- Agent run ends with "I was unable to..." after a single failed approach
- Tool calls per task is low (under 3) for tasks the user expects to be involved
- Final message includes phrases like "you could try", "I recommend you", "consider asking" without the agent having attempted those things
- User follow-up frequently rewords or pushes back ("did you try X?")

## Mitigations

### Explicit persistence instruction

System prompt addition that helps: "If a tool returns an error or unexpected result, try a different approach before reporting failure. You have N attempts; use them. Only report task failure if you have exhausted available approaches."

Specifying N (typically 3 to 5) matters. "Try a few times" is too vague.

### Tool inventory in the system prompt

Often the agent gives up because it forgot a tool exists. Restating the tool inventory at the start of each task helps:

```
You have access to:
- search(query): web search
- read(url): fetch a URL
- python(code): execute Python
- ask_user(question): ask the user for clarification

Use all of them as needed.
```

This is more useful than relying on the tool definitions alone, which can be buried in the context.

### Clarification before giving up

If the agent cannot proceed because something is ambiguous, the right behavior is usually to ask the user, not to give up. Add to the system prompt: "If you cannot proceed, ask the user a specific clarifying question before declaring failure."

### Different model for harder tasks

Smaller models sandbag more. If the task complexity is below the model's capability, do not over-rotate on this pattern; it is a sign you need a more capable model rather than more prompting.

### Logging giveup signals

Set up alerts for tasks that end with "I was unable to" or similar phrases. Sample 20 per week and check whether the agent actually exhausted reasonable approaches. The pattern is often visible quickly.

## When sandbagging is correct behavior

Sometimes giving up is right:

- The task requires authentication the agent does not have
- The user is asking for something the agent is correctly refusing
- The information genuinely does not exist or is not accessible
- Further attempts would have real cost (time, money, side effects) without enough upside

The distinction: legitimate giveup has a specific obstacle that the agent can articulate. Sandbagging has a vague "I cannot" without naming what specifically blocked it.

If the agent's failure explanation is more than 2 sentences and includes specific obstacles, it is probably legitimate. If it is one vague sentence, it is probably sandbagging.

## A note on user expectations

Users vary in how much agent persistence they want. A power user wants the agent to try 5 approaches; a non-technical user wants a quick answer or a quick "I cannot do that".

Calibrate per use case. Customer-support agents should attempt more before giving up than research-assistant agents. There is no universal right setting.
