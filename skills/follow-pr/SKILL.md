---
name: follow-pr
description: Follow an open GitHub PR until it is approved - wait for CI and reviews, fix failures and valid findings, reply to every comment, and keep the PR mergeable. Review findings go through review-loop's root-cause clusters, so a PR comment that repeats a locally fixed bug reopens that cluster. Use when the user asks to follow, babysit, or watch a PR until approval.
---

# Follow PR

Act as the author's agent on an open PR: fix what is valid, explain what is not, and keep the PR mergeable until it is approved. Treat every review finding and CI failure as a hypothesis to verify against the code and the intended behavior, not as an order.

Review findings are not triaged here. They go into the `review-loop` ledger and through `review-loop --ingest`, which clusters them with every earlier finding on the branch, local and PR, and fixes each cluster at its root cause. That keeps one history per branch, so a reviewer who finds a variant of an earlier bug gets a root-cause fix instead of another local patch.

Invoking this skill authorizes, for this PR only: pushing fixes to its branch, replying to and resolving review threads, and re-requesting review from reviewers whose feedback you addressed. The authorization ends when the loop stops or the user cancels or changes its scope.

## Inputs

`$follow-pr [<pr>]` (Codex syntax; Claude Code uses `/follow-pr`; from the `ship` plugin the name is `ship:follow-pr`; Pi uses `/skill:follow-pr`)

- `<pr>`: a PR number or URL. Default: the PR whose head is the current branch. If there is none, stop and say so.
- Write replies in the PR's language, which is the language of its title and body.

## Done when

All of these hold for the current head commit (`headRefOid`):

1. **Approved.**
   - `reviewDecision` is `APPROVED`. GitHub derives this from branch protection, including whether new pushes dismiss earlier approvals, so trust it when it is set.
   - If `reviewDecision` is empty because the repository requires no review, at least one reviewer other than the author has an `APPROVED` latest review, and no reviewer's latest review is `CHANGES_REQUESTED`.
   - Automated reviewers that re-review every push, such as Pullfrog, count only when the approving review's `commit_id` is the current head. A bot's approval of an earlier head is stale.
2. **Checks pass.** `gh pr checks --required` reports nothing failing or pending. Non-required failures caused by this PR get fixed as well.
3. **Feedback handled.** Every finding from every source has an evidence-backed disposition in the ledger and a reply on the PR. Sources include humans, bots, and other agents, in review bodies, inline threads, and standalone PR comments.
4. **Clean state.** The PR has no merge conflict with its base, the worktree is clean apart from the ledger, and everything is pushed.

For a draft PR, items 2 to 4 are enough: a draft is not ready for review, so stop once checks pass and no actionable finding remains. If the user asks you to wait for approval anyway, keep going.

Also stop when the PR is merged or closed (report that as such, not as success), when the user cancels or changes scope, or when a real blocker needs the user. Blockers include auth or permission errors, quota or service outages, and decisions only the user can make, such as a `blocked` finding. Waiting on a reviewer is not a blocker.

If nobody has been requested to review and no automation will request someone (CODEOWNERS, a review bot), tell the user once when CI is green, then keep waiting at the slow cadence.

## Each cycle

1. **Fetch the full state** in parallel with the commands below. Do not filter by author or review state: `COMMENTED` reviews and plain comments can contain blocking findings.
2. **CI failures.** Read the failed job logs before touching code (`gh run view <run-id> --log-failed`) and fix causes within the PR's scope. Re-run a job only when the logs show a transient failure, and only once. Never disable, skip, or weaken checks or tests to get green.
3. **Review feedback.** Compare the feedback with the ledger's `Source:` and `Reply:` fields to find what is new or still unresolved.
   - Record each new finding in the `review-loop` ledger as `P<cycle>-F<n>` with status `open`, its `Source:` (PR number, kind, comment URL, author), and the reviewer's claim as the scenario. The ledger path and format are in [../review-loop/references/local-ledger.md](../review-loop/references/local-ledger.md). One comment that raises several distinct problems becomes several findings. A finding that re-raises a dismissed or deferred one, with a new argument, is a new finding.
   - Run `review-loop --base <PR base> --ingest` (plugin name `ship:review-loop`; in Pi, read `../review-loop/SKILL.md` relative to this skill's directory) and follow it fully. It verifies, clusters, fixes, validates, and returns each finding's status and disposition. Do not fix review findings outside it, so every fix passes its closure gate.
   - Questions get a direct answer as a reply; record them in the ledger only when the answer reveals a problem. Praise and status messages from bots need nothing.
4. **Commit and push.** Commit only this PR's changes, never the ledger, following the repository's commit conventions. Push all of a cycle's fixes together, so CI and bots review one head instead of several.
5. **Respond.**
   - Reply to every handled finding with its disposition from the ledger: what changed and in which commit, why nothing changed, or the suggested follow-up. For a finding in a review body or a standalone comment, reply on the PR and link the source. Record each reply in the finding's `Reply:` field.
   - Resolve a bot's thread after your reply lands and the finding is `fixed` or `dismissed`. Reply to human reviewers but leave resolving to them, unless the repository's practice is for authors to resolve. Leave `blocked` and partially addressed threads open, and ask the user about `blocked` findings.
   - After addressing a reviewer's `CHANGES_REQUESTED`, re-request their review once (`gh pr edit <pr> --add-reviewer <login>`). For bots, use only the repository's existing re-review mechanism, such as an automatic review on push or a command already used in this PR's history. Do not guess commands.
   - Re-fetch afterwards to confirm the replies, resolutions, and review requests landed; exit codes alone are not proof.
6. **Resolve conflicts** by merging the base branch into the PR branch. Ask before rebasing or force-pushing.
7. **Check "Done when."** If the PR is not done and nothing is actionable, wait.

## Waiting

Use the waiting mechanism of the harness you are actually running in, judged from your available tools and session context, not from the model name. Read a tool's current schema or documentation before using it, and do not invent tool names or parameters. In order of preference:

1. **A PR watch or automation from the orchestrator that manages this session.** For example, T3 Code's `watch_pull_request` wakes you when checks finish, comments arrive, or the branch conflicts, so end the turn after calling it. An outer orchestrator takes precedence over the inner agent's scheduler: a Codex or Claude Code session running inside T3 Code uses T3 Code's watch.
2. **The agent's own scheduler.** For example, a thread-bound heartbeat automation in Codex, or in Claude Code `/loop`, scheduled wakeups, cron tools, or a background command that re-invokes you on exit, such as `gh pr checks <pr> --watch` for CI. Before creating an automation, look for an existing one for this PR and reuse it. After creating one, verify its ID, status, and schedule.
3. **In-session polling** as a last resort: sleep in interruptible chunks of 60 seconds or less, and fetch the PR state once per interval. Polling lasts only while the session is alive. If the session has to end, say that polling stopped; never claim a future check is scheduled when none is.

Wait about 5 minutes between checks while CI or a bot review is pending. When only human review remains, back off to 30 to 60 minutes. Notify the user only on meaningful changes: feedback handled, done, blocked, or input needed.

A scheduled wake may start with no memory of this run, so make its prompt self-contained: this skill's path, the PR URL, the repository or worktree path, whether the PR is a draft, and the instruction "Continue the follow-pr loop. Re-fetch the PR state first. When done, closed, cancelled, or blocked, stop and disable this automation."

When the loop stops, disable or pause any watch or automation you created, and verify that it is no longer active.

## Resume

After a wake, compaction, or restart, treat the PR, git, and the ledger as the source of truth, not memory:

- Re-read this file, and check the conversation for cancellation or scope changes.
- Inspect the worktree. Preserve uncommitted changes you cannot attribute to this run; if you cannot tell where to resume, stop and ask.
- Re-fetch the full PR state and reconcile it with the ledger's `Source:` and `Reply:` fields and your earlier commits. Verify each reply, resolution, push, and review request before repeating it.

## Report

When the loop stops: the PR URL and final state, the CI fixes, each review finding's disposition grouped by cluster with any recurrences of earlier clusters called out, open or `blocked` items, and the stop reason.

## Commands

`gh` fills in `{owner}` and `{repo}` from the current repository.

```sh
# Overview
gh pr view <pr> --json number,url,state,isDraft,author,headRefName,headRefOid,baseRefName,reviewDecision,mergeable,mergeStateStatus,reviewRequests,latestReviews

# Checks (exit code 8 = pending, 1 = failing or no checks)
gh pr checks <pr> --json name,state,bucket,link,workflow
gh pr checks <pr> --required
gh run view <run-id> --log-failed

# Reviews (with commit_id) and conversation comments
gh api --paginate 'repos/{owner}/{repo}/pulls/<pr>/reviews'
gh api --paginate 'repos/{owner}/{repo}/issues/<pr>/comments'

# Review threads
gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F pr=<pr> -f query='
query($owner:String!,$repo:String!,$pr:Int!,$endCursor:String){
  repository(owner:$owner,name:$repo){pullRequest(number:$pr){
    reviewThreads(first:100,after:$endCursor){
      pageInfo{hasNextPage endCursor}
      nodes{id isResolved isOutdated path line
        comments(first:100){nodes{author{login} body url createdAt originalCommit{oid}}}}}}}}'

# Reply to a thread (body read from a file to avoid quoting problems), then resolve it
gh api graphql -F id=<thread-id> -F body=@reply.md -f query='
mutation($id:ID!,$body:String!){addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$id,body:$body}){comment{url}}}'
gh api graphql -F id=<thread-id> -f query='
mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}'

# Reply on the PR conversation
gh pr comment <pr> --body-file reply.md
```
