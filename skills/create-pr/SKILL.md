---
name: create-pr
description: Create or update a GitHub pull request for the current work, then follow it through CI failures and review comments until it is approved. Creates the branch and commits transparently when the work is not committed yet. Use when the user asks to open, create, submit, or update a PR. Writes in English by default; `--ja` writes the PR in Japanese.
---

# Create PR

Turn the current work into a reviewable GitHub PR and stay with it until it is approved. The user should only have to ask for a PR: handle branching, committing, pushing, the description, and the follow-up yourself. Stop for input only in the cases listed under "When to ask".

## Inputs

Invocation syntax (Codex; in Claude Code and similar harnesses use `/create-pr` with the same arguments; from the `ship` plugin the name is `ship:create-pr`):

`$create-pr [--ja] [--draft] [--no-follow] [instructions]`

- `--ja`: write the PR title, body, and your PR comments and replies in Japanese. The default is English.
- `--draft`: open the PR as a draft, or convert an existing PR to a draft. Without this flag, open a regular PR and leave an existing PR's draft state alone.
- `--no-follow`: stop once the PR is created or updated and its URL is reported, skipping the follow-up in step 6.
- Free-form instructions, such as a base branch, an issue to link, reviewers, or scope, take precedence over the defaults below.

## 1. Inspect the current state

Gather these in parallel: `git status`, the current branch and its upstream, the remotes, the default branch (`gh repo view --json defaultBranchRef`), commits ahead of the likely base, and any existing PR for the branch (`gh pr view --json number,url,state,baseRefName,isDraft,body`).

## 2. Settle the base and the branch

**Base.** Use the existing PR's base if there is one. Otherwise use the branch the work forked from, determined from the upstream config, the reflog (`Created from ...`), or `git merge-base` against candidate branches. If the fork point cannot be determined uniquely, ask instead of guessing.

For a new PR whose base is not the default branch (a stacked PR), confirm the base with the user before pushing, unless they named it. A wrong base produces a confusing diff and can merge the work into the wrong branch. Creating the branch and committing locally can go ahead before that confirmation, since both are easy to undo.

**Branch.** The PR needs a dedicated branch with an ASCII name (lowercase letters, digits, `/`, `-`) that holds exactly this PR's commits.

- If you are on the base or default branch with uncommitted changes or unpushed commits, create a branch at the current HEAD with `git switch -c <name>`. Uncommitted changes carry over. If unpushed commits came along, reset the local base branch to its upstream (`git branch -f <base> <base>@{upstream}`) so those commits live only on the PR branch.
- Harnesses sometimes auto-create branch names that contain non-ASCII text from the task title. If such a branch has not been pushed, rename it with `git branch -m`. If it has been pushed and already has a PR, leave it and ask the user: renaming a PR's head branch on GitHub deletes the old head, which closes the PR.
- Name new branches after the repository's convention (look at recent branches and merged PRs). If there is no visible convention, use `<agent>/<short-kebab-summary>`, for example `codex/fix-login-redirect` or `claude/add-retry-backoff`.

## 3. Commit

- Read the diff and stage only the changes that belong to this PR. Leave unrelated changes uncommitted and mention them in your report. If it is unclear whether a change belongs, ask.
- Never commit secrets (`.env` files, keys, tokens) or generated artifacts that the repository doesn't track.
- Write commit messages that follow the repository's history (`git log`), including its format, such as Conventional Commits. When no convention is visible, write them in the PR's language. Add any commit trailers or attribution that your harness or organization requires.
- If a commit hook fails, fix the cause rather than bypassing it with `--no-verify`.
- If there are no commits ahead of the base and nothing to commit, stop and report that; don't open an empty PR.

If you haven't validated the change in this session, run the project's fast checks that cover it, such as lint, type checks, and related tests, before pushing. That's cheaper than a CI round trip. Record exactly what you ran and the result, because the PR body may only claim checks that actually ran.

## 4. Write the title and body

**Template.** Look for a PR template in `.github/`, `docs/`, or the repository root (`pull_request_template.md`, `PULL_REQUEST_TEMPLATE.md`, or a `PULL_REQUEST_TEMPLATE/` directory; names are case-insensitive). If there are several, pick the one that fits the change. Keep its headings, their order, and its checklist items. Follow instructions left in HTML comments. Check a box only when it is true, and write "N/A" under a section that doesn't apply instead of deleting it.

Add a section only when important information has no place in the template, such as testing done, breaking changes or migration steps, rollout risk, notes for reviewers, or known follow-ups. Put added sections after the template's own. Without a template, use Summary and Testing, plus other sections only as needed.

**Content.** Write for the reviewer:

- Lead with why the change is needed and what it does, at a level the reviewer can check against the diff.
- Point out non-obvious decisions and the places that need the closest look.
- List only the checks that were actually run, with their results.
- Link related issues (`Closes #123`) when the user, the branch name, or the commits identify one.
- Skip filler and restating the diff line by line.

**Title.** Match the style of recent merged PRs (`gh pr list --state merged --limit 20 --json title`), such as Conventional Commits or ticket prefixes. Otherwise write a short imperative summary.

**Language.** Write in English by default and in Japanese with `--ja`. In either language, keep template headings, code identifiers, commands, and log output verbatim; repository automation may parse the headings. Append any PR attribution lines that your harness or organization requires.

## 5. Push and create or update the PR

- Push with `git push -u origin <branch>`.
- **New PR:** `gh pr create --base <base> --head <branch> --title <title> --body-file <tmp-file>`, adding `--draft` only when the flag was given. Pass the body through a temporary file so shell quoting can't corrupt it.
- **Existing PR:** push any new commits and update the body with `gh pr edit --body-file <tmp-file>`. Leave the title, base, and draft state unchanged unless the user asked otherwise; with `--draft`, convert a ready PR to a draft with `gh pr ready <pr> --undo`. Never mark a draft ready for review unless the user asks. Read the current body first and keep what others added, such as checked boxes, links, and notes.
- Add reviewers, labels, or assignees only when the user asked for them. Repository automation such as CODEOWNERS handles the rest.
- Share the PR URL with the user as soon as the PR exists, before the follow-up starts.

## 6. Follow up until approved

Skip this step when `--no-follow` was given. Otherwise read [references/follow-up.md](references/follow-up.md) and run its loop on the PR. Invoking this skill authorizes that loop for this PR: pushing fixes, replying to and resolving review threads, and re-requesting review from reviewers whose feedback you addressed. The authorization ends when the loop stops or the user cancels or changes its scope.

For a draft PR, run the loop for CI and any feedback that arrives, but don't wait for approval: stop once checks pass and no actionable finding remains, since a draft isn't ready for review. If the user asks you to wait for approval anyway, keep going.

## When to ask

Ask the user only when:

- the base cannot be determined, or a new PR's base is not the default branch and the user didn't name it,
- it is unclear whether a change belongs in this PR,
- a pushed branch that has a PR would need renaming,
- a force-push would be needed to rewrite published history,
- review feedback requires a product or architecture decision beyond the PR's scope.

Otherwise decide, act, and explain what you did in the report.
