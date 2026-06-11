# User confusion as agent confusion

The user asks an unclear question. The agent does not ask for clarification; it interprets the question and proceeds. The interpretation is wrong, so the agent's whole task is wrong.

## What it looks like

User: "fix the bug in the order processing thing"

The agent has access to a codebase with multiple services that touch orders. There is no specific bug mentioned. The agent picks one service, scans for "obvious" issues, makes a change, and reports back.

The user meant a different service, or a different bug, or wanted the agent to ask. The work is wasted; the user is frustrated; trust drops.

## Why it happens

- Models are trained to be helpful; asking for clarification feels like deflection
- The agent guesses based on the most likely interpretation
- No explicit instruction to clarify when uncertain
- The cost of a wrong start is low to the agent (more tokens) but high to the user (their time)

## Detection signals

- User follow-up messages express frustration ("no, not that") or correction
- User restarts a task with a more specific request after the agent's initial output
- Agent's first step is highly specific (picked a service, picked a file) without confirmation
- The original user message is short and ambiguous

You can score this in user feedback: every "no", "that's not what I meant", "I meant", or task restart is a signal that the agent started without enough information.

## Mitigations

### Explicit clarification mandate

System prompt addition: "If the user request is ambiguous, ask one specific clarifying question before starting. Do not guess between equally likely interpretations. Asking is cheaper than redoing."

The word "specific" matters. A generic "could you clarify?" feels like deflection. "Do you mean the checkout service or the fulfillment service? Both touch order processing." is useful.

### Clarification decision rule

Help the agent know when to ask vs proceed. A rule of thumb:

- If there are 3+ reasonable interpretations and they would lead to different work: ask
- If there are 2 interpretations and one is clearly more common: assume the common one, state the assumption, proceed
- If there is 1 reasonable interpretation: proceed without comment

System prompt formulation:

```
Before starting a task:
1. If the request has 3+ reasonable interpretations, ask which one.
2. If it has 2, pick the more likely one and explicitly state your assumption.
3. If it has 1, proceed.
```

### Confirm assumptions before high-cost work

If the agent is about to spend significant resources (a long-running tool call, a write operation, a billable API), confirm the interpretation first:

```
Before I start: I understand you want me to [restate the task].
This will [describe what I will do]. Confirm to proceed?
```

For low-cost tasks, skip the confirmation; it adds friction. For high-cost tasks, the confirmation is worth one extra turn.

### Better question quality

If the agent is going to ask, the question should narrow effectively. Bad: "what do you mean?" Good: "do you want X, Y, or Z?" Multiple choice with concrete options closes faster than open-ended re-prompts.

### Show the interpretation before the work

Even when not asking, the agent can preview its interpretation:

```
I will: search the codebase for recent commits in the order service,
review them for potential bugs, and propose a fix.
Starting now.
```

If the user wanted something else, they can interrupt before the agent commits to a wrong path.

## When to skip clarification

Skip clarification when:

- The task is exploratory (the user wants to see what the agent does; they will iterate)
- The user is an experienced operator who prefers fast iteration to clarifications
- The cost of being wrong is low and feedback is cheap
- The user explicitly said "just do something"

The right clarification policy depends on the user, the cost, and the task. A one-size-fits-all approach over- or under-clarifies.

## A note on iterative users

Some users prefer to send a vague request, see what the agent does, then refine. For them, clarification adds friction.

For these users, the agent's first attempt is fine even if rough; the user expects to iterate. The system can detect this pattern by tracking how often the user accepts the first output vs corrects.

## A note on agent voice

Asking clarifying questions can be done well or badly. Badly: "I am unable to proceed without more information." Well: "Quick clarification: do you mean A or B?"

The first sounds like a refusal. The second sounds like a productive next step. Same intent, different perception.

System prompts should encourage the "productive next step" framing. Asking is a tool; it should feel like one.
