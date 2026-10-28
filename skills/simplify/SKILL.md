---
name: simplify
description: Simplify a recent diff when the user requests cleanup or clearer code, preserving observable behavior.
---

# Simplify Recent Code

Refine recent changes conservatively. Prefer readable, explicit code over fewer lines. Make no change when the code is already clear or when an improvement is subjective.

## Preserve behavior

- Preserve public APIs, data formats, errors, side effects, operation order, logging, and timing-sensitive behavior.
- Preserve external names and contracts unless the user explicitly requests a change.
- Preserve comments that explain why, legal notices, and tool directives. Remove only comments that restate clear code or became obsolete.
- Avoid speculative abstractions and unrelated cleanup.

## Determine scope

1. Read repository instructions and identify the current working tree.
2. Inspect staged, unstaged, and untracked changes. Treat them as the default scope.
3. When the working tree has no changes, inspect the latest commit.
4. Limit edits to changed code and directly adjacent code required for a coherent simplification.
5. Exclude generated files, vendored code, and snapshots unless explicitly requested.

If the scope cannot be determined safely, report the ambiguity instead of guessing broadly.

## Simplify conservatively

Prioritize improvements in this order:

1. Reduce unnecessary nesting, branching, and indirection.
2. Clarify names and control flow.
3. Remove redundant calculations, conversions, conditions, and dead local abstractions.
4. Consolidate related logic when doing so makes its intent clearer.
5. Replace nested ternaries with readable conditionals or a switch when appropriate.

Do not optimize for line count. Do not replace familiar code with clever syntax. Avoid large rewrites when a local change is sufficient.

## Evaluate commonization

Commonize duplicated logic only when all of the following hold:

- The occurrences represent the same concept or domain rule, not merely similar syntax.
- They should evolve together when that rule changes.
- The abstraction has a clear intent-revealing name.
- It reduces repetition without adding flags, callbacks, configuration objects, or branching to accommodate differences.
- The callers become at least as easy to understand as the duplicated code.

Prefer a local helper before introducing a shared module. Do not create vague utility abstractions or generalize for hypothetical reuse. Preserve duplication when the occurrences have different responsibilities, are likely to diverge, or the abstraction would increase coupling or obscure intent.

Treat occurrence count only as supporting evidence. Two copies of an important invariant may justify consolidation; three structurally similar blocks with different meanings may not. When uncertain, keep the duplication.

## Verify

1. Review the final diff for scope and unintended behavior changes.
2. Run the smallest relevant available checks, preferring targeted tests, then type checking and linting as appropriate to the repository.
3. Do not fix unrelated failures. Report failed or skipped checks and why.

## Report

Summarize meaningful simplifications and verification results. If no change materially improves the code, state that no simplification was needed.
