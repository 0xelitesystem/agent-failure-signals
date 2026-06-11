# Contributing

## What makes a good pattern entry

Each pattern must include:

1. What it looks like (concrete example from a real trace)
2. Why it happens (root causes)
3. Detection signals (mechanical or trace-pattern based)
4. Mitigations (at least 2, with trade-offs)
5. When the behavior is actually correct (negative space)

A pattern without detection signals is just an essay. A pattern without mitigations is just a complaint. Both parts matter.

## Submission

1. Add a new file in `patterns/` with the next number (`11-yourpattern.md`)
2. Update README.md table of contents
3. Open a PR

PR title format: `add: <pattern-name>` or `update: <pattern-name>: <change>`

## What gets rejected

- Patterns specific to a single proprietary framework without generalization
- Patterns about safety/alignment (different repo, different scope)
- Patterns that are just "the model got something wrong" without a recurring structure
- Long quotes from other people's writing without local synthesis

## What gets accepted

- Production failure patterns with at least 3 documented occurrences
- New mitigations for existing patterns
- Updated detection signals as tooling improves
- Cross-references between patterns (some failures are combinations)

## License

By contributing you agree your contribution is MIT-licensed.
