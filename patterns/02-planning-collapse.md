# Planning collapse

The agent makes a plan at the start of a task, then abandons it after the first or second step. Subsequent actions are reactive to immediate tool results rather than guided by the original plan.

## What it looks like

Step 1 of the trace: a detailed plan with 5 to 8 steps.

Steps 2 through N: each action responds to the previous tool result without reference to the plan. The agent never returns to "according to my plan, the next step is X".

Common variant: the model writes a plan, executes step 1 perfectly, then loses track of what step 2 was supposed to be and improvises.

## Why it happens

- The plan was generated once and never re-fed to the model in later turns; long contexts push it out of attention
- The model's planning capability is stronger than its plan-following capability (true for most models)
- Tool results dominate the prompt; the plan is buried near the top
- No structured representation of which step is current

## Detection signals

- Step 1 contains a numbered plan or "I will: 1...2...3..."
- Later steps do not reference the plan by step number
- Final action does not match what the plan said the last step would be
- Plan promises 7 steps; only 4 happen before the agent concludes

You can score this automatically: extract the planned step count from step 1, count actual steps, flag if the ratio is below 0.6.

## Mitigations

### Plan-then-execute with explicit state

After the planning step, store the plan as structured data. On each subsequent step, the system prompt or a tool result includes:

```
Plan:
1. [DONE] Get the customer ID from email
2. [IN PROGRESS] Look up customer's recent orders
3. Filter orders by status = pending
4. Summarize findings
5. Reply to user

Current step: 2
```

The model now sees the plan and current position every turn. Drift goes down sharply.

### Plan as a tool call

Define a `update_plan` or `set_current_step` tool. Force the agent to explicitly mark progression through the plan. Side effect: the trace shows the plan transitions, making collapse easy to spot.

### Smaller plans

Plans with 8+ steps collapse more often than plans with 3 to 4 steps. If a task requires 10 steps, decompose into nested plans (a 4-step master plan where each step is itself a 3-step sub-plan).

### Re-plan checkpoints

After every 3 tool calls, prompt the agent: "review your original plan, mark progress, adjust if needed". This is more expensive but rescues runs that would otherwise drift.

### Skip planning for simple tasks

If the task is 1 to 3 steps, skip the planning phase. Planning overhead for simple tasks invites collapse because the plan is more verbose than the execution.

## When planning collapse looks like correct adaptation

Sometimes abandoning the plan is right:

- Step 1 returned information that makes step 2 unnecessary
- A tool failed and the original plan's recovery path is wrong
- The user input changed mid-task

The distinction: a real adaptation says "step 2 is unnecessary because X; jumping to step 3". A collapse just stops referencing the plan without explanation.

Train your trace reviewers to spot the difference.

## A note on chain-of-thought

CoT prompts (visible reasoning before each action) reduce planning collapse because the model is forced to articulate the connection between current action and overall goal. The trade-off is more tokens and slower latency.

For tasks under 5 steps, skip CoT. For tasks over 5 steps or anything with multi-step error recovery, CoT pays back.

## A note on agentic frameworks

Most agentic frameworks (LangChain, LangGraph, AutoGen, custom loops) suffer from planning collapse at similar rates. The framework rarely fixes the issue; the prompt structure does.

A framework that does help: ones that maintain plan state as a first-class object passed to every step. If your framework treats the plan as just text that lives in the conversation history, drift is more likely.
