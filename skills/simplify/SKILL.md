---
name: simplify
description: Simplify the recent diff without changing observable behavior - delete what is not needed, reuse what already exists, and make the rest easier to read. Use when the user asks to simplify, clean up, or tidy recent changes.
---

# Simplify Recent Code

Make the recent change smaller and easier to read for the next person who maintains it, while its observable behavior stays exactly the same. Every line kept must be read, tested, and fixed later, so code that does not need to exist is the best thing to remove. Readability is the measure, not line count: a shorter form that takes longer to decode is not simpler. "No simplification needed" is a valid and common result.

## Scope

1. Read the repository instructions (`AGENTS.md`, `CLAUDE.md`, and similar).
2. Use what the user names: files, a commit, or a range. Otherwise use staged, unstaged, and untracked changes, or the latest commit when the working tree is clean.
3. Edit only the changed code and the adjacent code a coherent simplification needs. Read further, including callers, callees, tests, and nearby helpers, to judge each edit, but do not edit that code.
4. Skip generated files, vendored code, lockfiles, and snapshots unless the user asks.

If you cannot tell what belongs to the change, for example because unrelated edits are mixed in, report that instead of guessing broadly.

## What must not change

A simplification never changes what callers, users, or operators can observe:

- Public APIs, signatures, exported names, data formats, error types and messages, side effects and their order, logging, and timing-sensitive behavior.
- Validation at trust boundaries, error handling that prevents data loss, security checks, and accessibility. Code you move or merge keeps all of them.
- Comments that explain why, legal notices, and tool directives such as `eslint-disable` or `# type: ignore`. A `shortcut:` comment that names its limit records a decision; keep it and the code it describes.

When an improvement would require changing any of these, leave the code as it is and list the opportunity in the report.

## Find candidates

For each spot in scope, take the first option that applies:

1. **Delete.** Dead code; unused variables, parameters, options, and branches; flexibility nobody asked for; comments that restate the code or became stale; debug leftovers. Before calling something unused, search the whole tree, including tests, config, and string or dynamic references.
2. **Reuse.** The repository already has a helper, component, or pattern for this. Use it the way the surrounding code does.
3. **Built-in.** The standard library or platform already does what hand-written code does. When the project has its own helper for it, that wins. Never add a dependency.
4. **Straighten.** Flatten nesting with early returns, remove indirection that only forwards calls, drop redundant conversions, conditions, and recalculations, replace nested ternaries with plain conditionals, and rename local identifiers whose names mislead. Split a function only when it does unrelated jobs, never to make it shorter.
5. **Merge.** Consolidate duplicates only when the test below passes.

Leave a spot alone when the gain is a matter of taste, when the new form takes longer to understand, or when the only gain is fewer lines. Match the surrounding code's idiom, naming, and comment density rather than your own preferences. Prefer small local edits over a rewrite.

## Merge duplicates only when it clarifies

Commonize duplicated logic only when all of these hold:

- The copies express the same rule and must change together when that rule changes; looking alike is not enough.
- The shared code can have a name that states its intent.
- It needs no flags, callbacks, configuration, or branches to absorb differences between callers.
- Each caller reads at least as easily as before.

Prefer a local helper over a new shared module, and do not create a generic utility for hypothetical reuse. The number of copies is only supporting evidence: two copies of an important invariant can justify a merge, while three similar-looking blocks with different meanings do not. When in doubt, keep the duplication.

## Verify

1. Reread the final diff against the original. For each edit, confirm the old behavior still holds on the edge cases the old code handled: empty, zero, missing values, and error paths.
2. Run the smallest relevant checks the repository offers: targeted tests first, then type checking and linting.
3. When a check fails because of your edit, fix or revert that edit. Do not fix unrelated failures; report them.

## Report

- One line per meaningful simplification: what changed and why it is simpler, with `path:line`.
- Opportunities left alone because they would change behavior or a contract, if any.
- Checks run with their results, and any check skipped with the reason.
- `Not checked:` in one line, when something that mattered could not be verified.

When nothing improves the code, say `No simplification needed.` and what you checked in one line.
