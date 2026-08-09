# agent-failure-modes

The recurring ways LLM agents fail in production, with detection signals and mitigations for each.

This is for people shipping agents that use tools, run loops, or plan multi-step actions. Not a list of safety risks, not a catalog of jailbreaks. The patterns here are everyday operational failures that cost real users time and money.

## Contents

| Pattern | File |
|---|---|
| Tool-call loop | [patterns/01-tool-loop.md](patterns/01-tool-loop.md) |
| Planning collapse | [patterns/02-planning-collapse.md](patterns/02-planning-collapse.md) |
| Instruction drift | [patterns/03-instruction-drift.md](patterns/03-instruction-drift.md) |
| Sandbagging | [patterns/04-sandbagging.md](patterns/04-sandbagging.md) |
| Premature completion | [patterns/05-premature-completion.md](patterns/05-premature-completion.md) |
| Tool selection error | [patterns/06-tool-selection-error.md](patterns/06-tool-selection-error.md) |
| Argument hallucination | [patterns/07-argument-hallucination.md](patterns/07-argument-hallucination.md) |
| Error masking | [patterns/08-error-masking.md](patterns/08-error-masking.md) |
| Context window exhaustion | [patterns/09-context-exhaustion.md](patterns/09-context-exhaustion.md) |
| User confusion as agent confusion | [patterns/10-user-confusion-amplification.md](patterns/10-user-confusion-amplification.md) |

## How to use this

When an agent task fails:
1. Read the agent's full trace (every tool call, every response)
2. Find the step where things went wrong
3. Match the pattern from the list above
4. Apply the mitigation from that pattern's file

Most production agent failures match one of these 10 patterns. The remaining 10 to 20% are domain-specific.

## What this is not

- Not a safety guide. These are operational failures, not alignment issues.
- Not a benchmark. There is no scoring system here; agents fail in ways specific to the tools they have access to.
- Not exhaustive. New failure modes will appear as agents get more autonomous; this list will grow.

## A note on scope

Most patterns assume an agent loop that calls tools and processes results. Single-shot LLM use without tool calling has its own failure modes (covered in [rag-evaluation-rubrics](https://github.com/0xelitesystem/rag-evaluation-rubrics)).

Multi-agent systems (one LLM coordinating multiple LLM workers) have an additional set of failures around message passing and task decomposition; not covered here.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New patterns must include detection signals and at least one mitigation strategy.

## More

Part of a catalog of single-file browser tools and plain-language references, all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/). Built by [elitesystem.ai](https://elitesystem.ai).

## License

MIT.

## Related

- [agent-trace-viewer](https://github.com/0xelitesystem/agent-trace-viewer) - inspect agent traces to find these patterns
- [agentic-workflow-patterns](https://github.com/0xelitesystem/agentic-workflow-patterns) - design patterns to prevent these failures
- [ai-coding-prompt-recipes](https://github.com/0xelitesystem/ai-coding-prompt-recipes) - prompt structures that reduce agent failures in coding tasks
- [rag-evaluation-rubrics](https://github.com/0xelitesystem/rag-evaluation-rubrics) - eval methods for the generation step
- [vibe-coding-anti-patterns](https://github.com/0xelitesystem/vibe-coding-anti-patterns) - related anti-patterns in code-generating agents
- [prompt-injection-test-suite](https://github.com/0xelitesystem/prompt-injection-test-suite) - the safety side; covers attacks not in this list
