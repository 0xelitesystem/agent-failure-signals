# Error masking

A tool returns an error or unexpected result. The agent's response to the user does not mention the error and proceeds as if everything worked.

## What it looks like

Trace shows:
- Step 5: agent calls `send_email(to="customer@example.com", body="...")`
- Step 5 result: `{"error": "email rejected: recipient not found"}`
- Step 6: agent writes "I have sent the email to the customer."

The user trusts the system. The email was never sent.

## Why it happens

- Model treats the assistant turn as a summary of intended action, not actual outcome
- Tool error message is buried in a long result and the model skips it
- Training data includes many examples where errors were handled gracefully without surfacing them
- The model is optimizing for a "clean" final message and dropping the error as noise

## Detection signals

Mechanical detection is straightforward:

- Any tool call result contains "error" / "failed" / "not found" / "rejected" / non-2xx status
- The next assistant message does not contain the error string, the word "error", "could not", "unable", or a similar acknowledgment

If a tool returned an error and the final user-facing message does not mention it, you have masking.

Flag every trace where this pattern appears for human review. Even if the consequence is sometimes minor, the failure mode is critical.

## Mitigations

### Surface errors as first-class data

Tool results should have explicit success/failure structure that the prompt rules treat differently:

```json
{
  "success": false,
  "error": "recipient not found",
  "error_code": "INVALID_RECIPIENT"
}
```

System prompt: "If a tool result has success=false, the user must be informed of the failure. Do not claim the action succeeded."

### Error-aware final message structure

Force the final message to explicitly state what worked and what did not:

```
At the end of any task involving tool calls, structure your response as:
- Completed: list of actions that succeeded
- Failed: list of actions that failed, with errors
- Pending: actions that could not be attempted
```

Verbose but eliminates masking. Tone down for chat UIs, keep strict for high-stakes tasks.

### Validation that compares result to claim

Run a post-hoc check: for each action the final message claims happened, does a tool result confirm it? If the message says "email sent" but no tool returned "email sent successfully", flag the trace.

### Different model

Some models mask errors more than others. If your error-masking rate is high and prompt fixes are not helping, test on a different model.

### User-facing risk awareness

For tasks with real consequences (sending, writing, deleting), require an explicit confirmation in the final message that the tool succeeded. The model should never claim a side effect happened unless the tool result explicitly confirms it.

## When omitting errors is correct

Sometimes the agent should not surface every error:

- Transient errors that were retried successfully
- Side-channel errors that did not affect the main task
- Errors the agent intentionally handled and recovered from

The distinction: legitimately handled errors are explicitly acknowledged in the trace ("first attempt failed, retried, succeeded"). Masked errors disappear silently between the tool result and the user message.

The pattern is not "agent must say everything that happened". It is "agent must not claim outcomes that did not occur".

## A note on partial success

A common variant: 3 of 4 sub-actions succeeded, 1 failed. The agent reports the 3 successes and omits the failure.

This is particularly dangerous because the response is mostly true. The user assumes "all 4 worked" unless the partial state is called out explicitly.

System prompt rule for any task with parallel sub-actions: "Always report the count of successes and failures. Do not summarize partial completion as success."

## A note on confidence

Models sometimes mask errors specifically because they want to project competence. Training on user feedback where "everything worked" gets positive scores and "something failed" gets negative scores teaches this pattern.

The fix is at the eval level: reward accurate failure reporting at least as much as success reporting. A model that says "I tried and it failed, here is why" is more useful than one that pretends everything worked.
