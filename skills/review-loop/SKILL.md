---
name: review-loop
description: Run a requested bounded local review-fix-review loop, with independent reviews and no GitHub mutations.
---

# Review Loop

Drive a local change to review convergence without creating or updating a pull request. Treat every finding as a hypothesis: verify it against the code and intended behavior before editing.

## Inputs

Invocation syntax (Codex; in Claude Code and Cursor CLI use `/review-loop` with the same arguments; from the `ship` plugin the name is `ship:review-loop`):

`$review-loop [--base <ref> | --uncommitted] [--max <rounds>]`

- Accept `--max 5` and `--max=5`. Default to `5`; require a positive integer.
- `--base` reviews the branch-shaped diff that would merge into the locally available comparison ref. Resolve the comparison ref to its locally available upstream when that upstream exists and is ahead of the local branch; otherwise use the local branch. If the local branch cannot be resolved, try its configured upstream. Compute `git merge-base HEAD <comparison-ref>` and review `git diff <merge-base-sha>`; do not fetch or silently substitute another target.
- `--uncommitted` reviews staged, unstaged, and untracked changes.
- With no target flag, use a locally resolvable remote default branch for a non-default current branch; otherwise use uncommitted changes. Do not fetch merely to discover a base. Ask only when the target remains ambiguous.
- A round is one completed independent review pass. Never exceed `--max`.
- Reserve the last allowed pass for verification. Run it only after every known cluster passes the closure gate and required validation passes. If it returns actionable findings, stop at the limit without claiming convergence.

## Pluggable review execution

Select the review backend from the active agent harness, not the model name. Record the harness, backend, reviewer model and effort, and exact target in the ledger and reuse that choice during the run unless the user changes it.

Run every pass in a fresh subagent, or the backend's isolated equivalent, so the review context stays separate from the fixing agent's. The reviewer must use the main agent's model and reasoning effort. Set both explicitly whenever the delegation interface accepts them, because harness defaults such as a configured default subagent model, an agent definition, or CLI configuration can differ. If either cannot be set or confirmed, report that and ask before the first pass instead of running on another model or effort. Wait for the reviewer's final response before triaging, editing, or starting another pass, then continue the loop until a stop condition applies; a dispatched reviewer is not a completed pass.

- **Codex:** use a fresh delegated subagent with `$review-agent` for every review pass. This explicitly authorizes review delegation when subagents are available. If delegation is unavailable, use `codex review --base <ref>` or `codex review --uncommitted` from the repository root, subject to environment approvals. A wait timeout or missing progress message does not mean delegation is unavailable. Verify that the command reviews the resolved target below; do not silently accept a different comparison ref.
- **OpenCode:** use the `/review` command for every pass. Inspect the installed command definition and supported invocation interface first, then pass the exact resolved target and review criteria using that interface. `/review` is a harness command, not a shell executable; do not invent flags or replace it with Codex review. Execute it in a fresh review context or isolated subtask/session so the fixing agent does not review its own work in its existing context. If the harness cannot dispatch `/review` or isolate the pass, report the limitation and ask how to run it rather than counting a self-review as a completed pass.
- **Claude Code:** run each pass in exactly one fresh `general-purpose` subagent through the Agent tool with `run_in_background: false`. Pass `model` as the alias of the main agent's model family, taken from the model ID in its system prompt (for example `opus`), and `effort` as the main agent's active effort level from `echo $CLAUDE_EFFORT` (omit `effort` only when that is unset). Do not omit `model`: the subagent can then fall back to a different default model. Do not use `fork`: it inherits the fixing agent's context. In the prompt, have the subagent invoke `/code-review medium` through the Skill tool with the exact resolved target and review criteria, keep the `medium` level, invent no flags (never `--fix` or `--comment`), pass the same `model` and `effort` to any agent the review recipe launches, and end with every finding or an explicit no-findings result plus its own exact model ID and `echo $CLAUDE_EFFORT` output, because only its final message reaches the main agent. Accept the pass only when both match the main agent's. If the Agent tool or `/code-review` is unavailable, or the model, effort, or contract below cannot be honored, report the limitation and ask for an alternative; do not silently substitute another reviewer, model, or effort, or count a self-review as a completed pass.
- **Cursor CLI:** follow the `/review-bugbot` launch instructions for every review pass: launch exactly one fresh `bugbot` subagent with `description: "Bugbot"` and `run_in_background: false`, using its fixed prompt shape. Set `Full Repository Path` to the repository root. For `--uncommitted`, use `Diff: uncommitted changes`. For `--base`, use `Diff: branch changes` with `Base Branch: <resolved comparison ref>`. Put the fingerprint, `.review/` exclusion, read-only contract, cluster context from the contract below, and any one-off user criteria in `Custom Instructions`. Skip its branch-checkout step; the loop reviews the current checkout. Its no-fix and no-rerun rules apply to the subagent only; this loop still triages, fixes, and runs later passes. Apply its single retry on invocation failure, then report the limitation and ask for an alternative; do not substitute `/review-security`, another reviewer, or a self-review.
- **Other harnesses:** use the review backend explicitly configured by the user, applicable repository instructions, or the harness. If none is defined, ask the user which reviewer or review command to use before starting a review pass. Do not guess a provider or silently default to Codex. If a configured backend is unavailable or cannot meet the contract below, explain the limitation and ask for an alternative. Preserve the choice across continuation; do not ask again without a material change.

Every backend must follow the same contract: inspect the complete resolved diff, applicable `AGENTS.md`, surrounding code, tests, and call sites; return actionable findings or an explicit no-findings result; and make no file changes or external mutations. Give it the exact target, fingerprint, ledger exclusion, and any one-off user criteria. From the second pass on, also give it the closed clusters' invariants to re-check for bypasses and the dismissed signatures with their evidence, asking it to report a dismissed item again whenever that evidence does not hold for the current code; these add checks and never narrow the review. Exclude `.review/` from every pass. For uncommitted changes, include staged, unstaged, and untracked files. For a base review, use the resolved comparison ref and merge base from Inputs. Verify what was actually reviewed before accepting the result; a failed invocation or incomplete review is not a completed round.

Do not substitute a self-review in the fixing agent's existing context. GitHub comments, `gh`, commits, pushes, and network fetches are not part of this workflow. The backend changes only how a review is executed; round limits, fingerprints, cluster closure, validation, and convergence rules remain the same.

## Reviewer lifecycle and waiting

These rules apply to every backend selected above, whether it runs as a delegated agent, an isolated harness command/session, or a CLI process. Use the selected backend's actual lifecycle interface; do not assume another harness's tools or notification behavior.

- Before waiting, record the reviewer/session or process identifier, target fingerprint, and state (`running`, `completed`, `failed`, or `interrupted`) in the ledger. Keep one active reviewer for that pass; do not change reviewed inputs while it runs.
- Treat wait timeouts and absent intermediate progress reports as inconclusive. Progress delivery depends on the harness and selected backend; missing notifications do not establish inactivity. Repeated empty waits, elapsed time, and a silent notification channel alone never justify declaring the review stalled, interrupting it, launching a replacement, or switching backends.
- Use the harness's supported status/progress interface to distinguish running work from completion or failure. If needed and accessible, inspect only that reviewer's identified local session log read-only; do not guess log paths or scan unrelated sessions. Record observed activity separately from inferred state. If progress cannot be observed, report that limitation and keep waiting; do not convert unknown progress into failure. Use bounded waits and send concise user updates at least every 60 seconds without claiming unseen progress.
- Interrupt only for an explicit user stop, an invalidated target, a contract violation, or confirmed execution failure/blockage that cannot proceed. Record the concrete evidence and reason. Before replacing a reviewer, confirm the original has stopped. Backend fallback requires established unavailability or an unrecoverable failure, and all applicable approvals; silence is not authorization to use another execution or disclosure path.
- After interruption or compaction, recover the recorded reviewer identifier and check its state before dispatching another review. If it is still running on the same fingerprint, wait for that pass. If a final report is available, verify coverage and fingerprint before accepting it. An incomplete or interrupted attempt consumes no completed round; a resumed attempt remains the same pass. Start a fresh reviewer for the next pass only after the current pass has ended.

**Codex delegation note:** `wait_agent` may not forward a child's commentary or tool activity. Check the delegated agent's supported status interface, or its identified local session log when accessible, before interpreting an empty wait. This note does not prescribe `wait_agent` or `$review-agent` for other harnesses.

**Claude Code note:** a foreground Agent call returns only after the subagent finishes, so the call itself is the wait; record the returned agent ID and final state. If the reviewer runs in the background anyway, wait for its completion notification without changing reviewed inputs or launching another reviewer.

## Local ledger

Read [references/local-ledger.md](references/local-ledger.md) when starting or resuming the loop. Preserve its history, exclude it from reviews, and never stage or commit it.

## Start and resume

1. Read applicable `AGENTS.md`, inspect `git status`, current branch, HEAD, and the complete target diff. Preserve unrelated changes.
2. Resolve the target and ledger path. If the ledger exists, reconstruct open findings, earlier fixes, and historical root-cause clusters from it.
3. Partition a cross-cutting target by state owner or lifecycle coordinator. Every changed subsystem must pass its cluster gates before the verification pass. If the subsystems cannot be reasoned about independently, recommend splitting the change instead of consuming review rounds.
4. Record a content fingerprint for the exact reviewed inputs, excluding the ledger. Include HEAD, the merge base for `--base`, and hashes covering the reviewed diff and untracked files when applicable. A review result is valid only for that fingerprint.
5. After interruption, compaction, or uncertainty, reread the ledger and recompute the fingerprint. Resume from the earliest incomplete step. If existing changes cannot be attributed safely, stop instead of overwriting them. Repeating a read-only review is safe; repeating an uncertain edit is not.

## Triage before editing

Cluster every finding with the algorithm in [references/cluster-closure.md](references/cluster-closure.md) before editing anything. In short:

1. **Signature.** Record each finding's trigger, lifecycle phase, persistence sink, effect, and a signature `<state owner> / <invariant> @ <enforcement boundary>`. State the invariant as a property the state must keep, not as the symptom or the cited line.
2. **Strong match, then weak.** The same invariant on the same state owner joins one cluster, with its boundary `?` until the boundary shared by all members is found. A shared phase, state owner, sink, file, or symptom alone only proposes a merge, accepted when one hypothesis and one fix boundary explain both.
3. **History.** Apply the same test to every historical cluster and dismissal, by meaning rather than wording. A cluster match is a recurrence at any boundary. A dismissal match is re-dismissed only after re-triage from scratch against the current code, including any new argument from the reviewer.
4. **Split or merge.** Keep a cluster only when its hypothesis predicts every member and names one fix boundary; otherwise split it. Merge clusters only when one hypothesis predicts both and one fix boundary covers both.
5. **Predict siblings.** Search the target diff and call graph for unreported violations of each cluster's invariant; verify each one, then fix or dismiss it before closure.
6. **Order.** Fix upstream boundaries first, then by severity and recurrence; re-triage downstream clusters after each upstream fix.

For every cluster, trace one complete path from trigger through persistence and retry or cleanup, inspect every caller and sibling path crossing the boundary, and write a root-cause hypothesis before choosing a fix. Enter stabilization mode on any trigger listed in the reference, including two current findings in one cluster, a historical match, or an accepted weak-signal merge. A singleton cluster that meets none of them takes the fast path there instead of the full inventory.

## Cluster closure

Before treating a cluster as fixed, establish the shared cause, classify affected callers and persistence sinks,
enforce the invariant at the shared publish boundary, cover the trigger and relevant sibling/recovery paths, and
check for bypasses. Read [references/cluster-closure.md](references/cluster-closure.md) for the full closure gate when
triaging a valid finding, especially recurrence, asynchronous lifecycle changes, or identifier/ownership changes.
Do not run another review while a known cluster remains open. Keep the detailed inventory conditional on stabilization.

## Each round

1. Recompute the target fingerprint, then run one fresh independent review. Record the fingerprint and every actionable finding before editing.
2. Reconcile findings with the ledger and root-cause clusters. Validate them against intended behavior, all callers, sibling paths, tests, and the complete diff. Dismiss false positives with concrete evidence.
3. For every valid cluster, complete any required inventory, then fix the narrowest shared boundary that restores the invariant. Avoid per-caller guards unless the contracts intentionally differ.
4. Add or update the smallest regression scenarios that satisfy the closure gate without duplicating equivalent tests.
5. Apply the cluster closure gate over the complete target diff and relevant call graph. If it fails, investigate instead of stacking another patch or starting a review.
6. Run focused validation and practical repository-required checks. Record commands and results in the ledger.
7. Reinspect the diff, recompute the fingerprint, and update cluster and subsystem dispositions. Record the round's convergence signals (reference) and act on them before the next pass. Start another review only when every known cluster is closed or concretely dismissed. Do not commit or push unless the user separately asks.

## Recurrence means root-cause mode

When a new finding belongs to an earlier cluster, reopen that cluster and mark the earlier fix incomplete. Compare the current implementation and complete target diff with the ledger's earlier enforcement boundary and determine why it was narrower than the invariant. Do not repeat the same leaf-level fix under another caller or condition.

Trace the end-to-end lifecycle and shared state or contract, replace repeated guards with the smallest shared fix, and remove local patches made redundant by it. Cover both the earlier and current symptoms. Do not run another review until the reopened cluster passes the closure gate. If the root fix requires a product or architecture decision outside scope, leave the cluster open and ask the user rather than spending more rounds.

## Stop conditions

Repair validation failures caused by the requested changes and rerun affected checks within the authorized scope. A failed check prevents convergence; it does not by itself end repair work.

Declare convergence only when a fresh independent review after the latest edit returns no actionable findings for the current fingerprint, every subsystem and cluster is closed (superseded or dismissed counts as closed), and required validation passes. A re-reported dismissal that still holds after re-triage from scratch, including the reviewer's new argument, with no edit, is not actionable; list it in the final report. The ledger alone, an earlier clean review, or unchanged filenames are not proof.

Stop without claiming convergence when the round limit is reached with findings, validation is blocked beyond an in-scope repair, the target changes unexpectedly, a fix requires a product decision outside the requested scope, or the convergence signals show the target is too wide (then recommend how to split it). Report rounds used, fixes, dismissals, remaining findings, validation, target HEAD/worktree state, ledger path, and the exact stop reason.
