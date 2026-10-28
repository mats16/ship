# Following a PR to approval

Act as the author's agent: fix what is valid, explain what is not, and keep the PR mergeable until it is approved. Treat every review finding and CI failure as a hypothesis to verify against the code and the intended behavior, not as an order.

## Done when

All of these hold for the current head commit (`headRefOid`):

1. **Approved.**
   - `reviewDecision` is `APPROVED`. GitHub derives this from branch protection, including whether new pushes dismiss earlier approvals, so trust it when it is set.
   - If `reviewDecision` is empty because the repository requires no review, at least one reviewer other than the author has an `APPROVED` latest review, and no reviewer's latest review is `CHANGES_REQUESTED`.
   - Automated reviewers that re-review every push, such as Pullfrog, count only when the approving review's `commit_id` is the current head. A bot's approval of an earlier head is stale.
2. **Checks pass.** `gh pr checks --required` reports nothing failing or pending. Non-required failures caused by this PR get fixed as well.
3. **Feedback handled.** Every finding from every source has an evidence-backed disposition. Sources include humans, bots, and other agents, in review bodies, inline threads, and standalone PR comments.
4. **Clean state.** The PR has no merge conflict with its base, the worktree is clean, and everything is pushed.

For a draft PR, items 2 through 4 are enough.

Also stop when the PR is merged or closed (report that as such, not as success), when the user cancels or changes scope, or when a real blocker needs the user. Blockers include auth or permission errors, quota or service outages, and decisions only the user can make. Waiting on a reviewer is not a blocker.

If nobody has been requested to review and no automation will request someone (CODEOWNERS, a review bot), tell the user once when CI is green. Then keep waiting at the slow cadence.

## Each cycle

1. **Fetch the full state** in parallel with the commands below. Don't filter by author or review state, because `COMMENTED` reviews and plain comments can contain blocking findings.
2. **Triage** everything new or still unresolved:
   - Read the code and intent before changing anything. A valid in-scope finding gets fixed. One that is invalid, obsolete, or already addressed gets an explanation with evidence. One that is out of scope or pre-existing gets an explanation and a suggested follow-up; it doesn't expand the PR. If you are uncertain, leave the thread open and say why.
   - For CI failures, read the failed job logs before touching code (`gh run view <run-id> --log-failed`) and fix causes that fall within the PR's scope. Re-run a job only when the logs show a transient failure, and only once. Never disable, skip, or weaken checks or tests to get green.
   - Group findings that share a root cause, such as the same invariant, enforcement boundary, or lifecycle step, even when they point at different lines. Fix the shared cause once at the narrowest boundary common to all affected paths, and check the sibling callers. A finding that recurs after an earlier fix means that fix was too local: reopen it and fix the shared cause instead of adding another guard.
3. **Fix, validate, push.** Run focused validation for each fix: reproduce the failing check locally if you can, and run the related tests. Commit only this PR's changes and push all of a cycle's fixes together, so CI and bots review one head instead of several.
4. **Respond.**
   - Reply to every handled thread with its disposition: what changed and in which commit, or why nothing changed. For findings in a review body or a standalone comment, reply on the PR and link the source.
   - Resolve a bot's thread after your reply lands and the finding is addressed or dismissed. Reply to human reviewers but leave resolving to them, unless the repository's practice is for authors to resolve. Leave uncertain or partially addressed threads open.
   - After addressing a reviewer's `CHANGES_REQUESTED`, re-request their review once (`gh pr edit <pr> --add-reviewer <login>`). For bots, use only the repository's existing re-review mechanism, such as an automatic review on push or a command already used in this PR's history. Don't guess commands.
   - Write replies in the PR's language. Re-fetch afterwards to confirm the replies and resolutions landed; exit codes alone aren't proof.
5. **Resolve conflicts** by merging the base branch into the PR branch. Ask before rebasing or force-pushing.
6. **Check "Done when."** If the PR isn't done and nothing is actionable, wait as described below.

## Waiting

Use the waiting mechanism of the harness you are actually running in, and judge that from your available tools and session context, not from the model name. Read a tool's current schema or documentation before using it, and don't invent tool names or parameters. In order of preference:

1. **A PR watch or automation from the orchestrator that manages this session.** Examples include T3 Code's `watch_pull_request` (it wakes you when checks finish, comments arrive, or the branch conflicts, so end the turn after calling it). An outer orchestrator takes precedence over the inner agent's scheduler; a Codex or Claude Code session running inside T3 Code uses T3 Code's watch.
2. **The agent's own scheduler.** Examples include a thread-bound heartbeat automation in Codex, and in Claude Code `/loop`, scheduled wakeups, cron tools, or a background command that re-invokes you on exit, such as `gh pr checks <pr> --watch` for CI. Before creating an automation, look for an existing one for this PR and reuse it. After creating one, verify its ID, status, and schedule.
3. **In-session polling** as a last resort: sleep in interruptible chunks of 60 seconds or less, and fetch PR state once per interval. Polling lasts only while the session is alive. If the session has to end, say that polling stopped; never claim a future check is scheduled when none is.

Wait about 5 minutes between checks while CI or a bot review is pending. When only human review remains, back off to 30–60 minutes. Notify the user only on meaningful changes: feedback handled, done, blocked, or input needed.

A scheduled wake may start with no memory of this run, so make its prompt self-contained. Include this skill's path, the PR URL, the repository or worktree path, the language (`--ja` or not), whether the PR is a draft, and the instruction: "Continue the create-pr follow-up loop. Re-fetch the PR state first. When done, closed, cancelled, or blocked, stop and disable this automation."

When the loop stops, disable or pause any watch or automation you created, and verify that it is no longer active.

## Resume

After a wake, compaction, or restart, treat the PR and git as the source of truth, not memory:

- Re-read this file, and check the conversation for cancellation or scope changes.
- Inspect the worktree. Preserve uncommitted changes you cannot attribute to this run; if you can't tell where to resume, stop and ask.
- Re-fetch the full PR state, and reconstruct what has been handled from your earlier replies and commits. Verify each reply, resolution, push, and review request before repeating it.
- If the language wasn't recorded, infer it from the PR body.

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
