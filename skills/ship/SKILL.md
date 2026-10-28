---
name: ship
description: Finish the current code change and open a PR in one run - simplify the diff, drive a local review-fix-review loop to convergence, create the PR, then follow it to approval. Use when the user asks to ship, wrap up, or finish a change into a PR.
disable-model-invocation: true
---

# Ship

Run `simplify`, `review-loop`, `create-pr`, and `follow-pr` in that order on the current work. Each step follows its own skill; this skill only fixes the order, the targets, and the gates between them. Invoke each one as a skill (Claude Code: the Skill tool; Codex: `$<name>`) and follow it fully. When this skill comes from the `ship` plugin, use the plugin names `ship:simplify`, `ship:review-loop`, `ship:create-pr`, and `ship:follow-pr`, since a bare `simplify` may resolve to a different skill; when the skills are installed directly, use the bare names. Do not reimplement them here.

## Inputs

`/ship:ship [--ja] [--draft] [--no-follow] [--max <rounds>] [instructions]` (Codex: `$ship:ship ...`; installed directly: `/ship` or `$ship`)

- `--max` goes to `review-loop`. Its default applies when omitted.
- `--ja`, `--draft`, and the free-form instructions go to `create-pr`.
- `--no-follow` skips step 4.

## 0. Check there is something to ship

Inspect `git status`, the current branch, and commits ahead of the likely base. If there are no changes and no commits ahead, stop and say so.

## 1. Simplify

Run `simplify` on the default scope (staged, unstaged, and untracked changes). Fix any check it breaks before moving on. "No simplification needed" is a valid result.

## 2. Review loop

Run `review-loop` with a target that covers everything the PR will contain:

- On the default branch, or with no commits ahead of the base: `--uncommitted`.
- On a branch that already has commits ahead of the base: `--base <base>`. First run `git add -N` on the new untracked files that belong to this change, so the base diff includes them.

Gate: continue to step 3 only when `review-loop` declares convergence. Otherwise stop, report what it reported (rounds, remaining findings, stop reason, and any recommended reframe), and ask whether to apply the reframe or open the PR anyway.

## 3. Create the PR

Run `create-pr` with the passed-through flags and instructions. Validation from steps 1 and 2 counts as run in this session; list those exact commands and results in the PR body instead of rerunning them, unless the code changed afterwards.

## 4. Follow the PR

Skip this step with `--no-follow`. Otherwise run `follow-pr` on the PR from step 3. PR review findings land in the same ledger as step 2's, so a reviewer who finds a variant of a locally fixed bug reopens that cluster.

## Report

One short section per step: what `simplify` changed, the `review-loop` result (rounds, fixes, dismissals), the PR URL, and the `follow-pr` result (final state, findings handled, recurrences of local clusters).
