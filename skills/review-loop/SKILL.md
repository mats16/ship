---
name: review-loop
description: Drive a local change to review convergence with a bounded review-fix-review loop - fresh independent reviewer passes, findings clustered by root cause, and no GitHub mutations. Use when the user asks for a review loop. Its `--ingest` mode triages PR review findings that `follow-pr` records, through the same clusters.
---

# Review Loop

Drive a local change until a fresh, independent review of the current code finds nothing actionable. Two ideas carry the loop:

- **Independence.** The agent that fixes the code cannot see its own blind spots, so every pass runs in a fresh reviewer context on the same model and effort as the main agent.
- **Root causes, not symptoms.** Findings are clustered by the invariant they violate. A cluster closes only when that invariant holds at the boundary every violating path crosses, so the next pass does not rediscover the same bug one caller at a time. A finding that matches an earlier cluster means the earlier fix was too narrow.

Treat every finding as a hypothesis: verify it against the code and the intended behavior before editing. This skill never touches GitHub: no `gh`, comments, commits, pushes, or network fetches. Commit or push only when the user separately asks.

## Inputs

`$review-loop [--base <ref> | --uncommitted] [--max <rounds>] [--ingest]` (Codex syntax; Claude Code and Cursor CLI use `/review-loop` with the same arguments; from the `ship` plugin the name is `ship:review-loop`; Pi uses `/skill:review-loop`)

- `--max`: accept `--max 5` and `--max=5`. A positive integer, default `5`. A round is one completed independent review pass; never exceed `--max`.
- `--base <ref>`: review the diff this branch would merge into `<ref>`. Compare against `<ref>`'s locally available upstream when that exists and is ahead of the local branch, otherwise against the local branch; if the local branch cannot be resolved, try its configured upstream. Review `git diff <merge-base-sha>`, where the merge base is `git merge-base HEAD <comparison-ref>`. Do not fetch, and do not substitute another target.
- `--uncommitted`: review staged, unstaged, and untracked changes.
- No target flag: on a non-default branch with a locally resolvable remote default branch, use that branch as `--base`; otherwise use `--uncommitted`. Do not fetch merely to discover a base. Ask only when the target stays ambiguous.
- `--ingest`: run no review pass. Triage and fix the external findings already recorded in the ledger, as described in [Ingest external findings](#ingest-external-findings).

## Review backend

Choose the backend from the harness you are running in, not from the model name. Record the harness, backend, reviewer model and effort, and exact target in the ledger, and keep that choice for the whole run unless the user changes it.

Every pass runs in a fresh subagent, or the backend's isolated equivalent, on the main agent's model and reasoning effort. Set both explicitly whenever the interface accepts them, because harness defaults such as a configured default subagent model, an agent definition, or CLI configuration can differ. If either cannot be set or confirmed, report that and ask before the first pass. A dispatched reviewer is not a completed pass: wait for its final response before triaging, editing, or starting another pass, then continue the loop until a stop condition applies.

- **Codex:** run every pass as a fresh delegated subagent with `$review-agent`. This explicitly authorizes review delegation when subagents are available. A wait timeout or a missing progress message does not mean delegation is unavailable. If delegation is unavailable, use `codex review --base <ref>` or `codex review --uncommitted` from the repository root, subject to environment approvals, and verify that it reviews the resolved target, not a different comparison ref.
- **OpenCode:** run the `/review` command for every pass. It is a harness command, not a shell executable: inspect its installed definition and invocation interface first, pass the exact resolved target and review criteria through that interface, and invent no flags. Run it in a fresh review context or isolated subtask or session.
- **Claude Code:** run each pass in exactly one fresh `general-purpose` subagent through the Agent tool with `run_in_background: false`. Pass `model` as the alias of the main agent's model family, taken from the model ID in its system prompt (for example `opus`); without it the subagent can fall back to a different default model. Pass `effort` as the main agent's effort level from `echo $CLAUDE_EFFORT`, omitting it only when that is unset. Do not use `fork`, which inherits the fixing agent's context. In the prompt, have the subagent:
  - invoke `/code-review medium` through the Skill tool with the exact resolved target and review criteria, keeping the `medium` level and inventing no flags (never `--fix` or `--comment`);
  - pass the same `model` and `effort` to any agent the review recipe launches;
  - end with every finding or an explicit no-findings result, plus its own exact model ID and its `echo $CLAUDE_EFFORT` output, because only its final message reaches the main agent.

  Accept the pass only when both match the main agent's.
- **Cursor CLI:** follow the `/review-bugbot` launch instructions for every pass: exactly one fresh `bugbot` subagent with `description: "Bugbot"` and `run_in_background: false`, using its fixed prompt shape. Set `Full Repository Path` to the repository root. For `--uncommitted`, use `Diff: uncommitted changes`; for `--base`, use `Diff: branch changes` with `Base Branch: <resolved comparison ref>`. Put the fingerprint, the `.review/` exclusion, the read-only contract, the cluster context from the contract below, and any one-off user criteria in `Custom Instructions`. Skip its branch-checkout step, since the loop reviews the current checkout. Its no-fix and no-rerun rules bind the subagent only; this loop still triages, fixes, and runs later passes. Apply its single retry on invocation failure. Do not substitute `/review-security`.
- **Other harnesses:** use the review backend that the user, the repository instructions, or the harness configures. If none is configured, ask which reviewer or review command to use before the first pass. Do not guess a provider or default to Codex. Keep the choice across continuation; ask again only after a material change.

When a backend is unavailable or cannot meet the contract below, report the limitation and ask for an alternative. Never substitute another reviewer, model, or effort, and never count a self-review in the fixing agent's context as a pass. The backend changes only how a review runs; round limits, fingerprints, clustering, validation, and convergence rules stay the same.

### Reviewer contract

Every backend inspects the complete resolved diff, the applicable `AGENTS.md`, surrounding code, tests, and call sites; returns actionable findings or an explicit no-findings result; and makes no file changes or external mutations.

Give the reviewer the exact target, the fingerprint, the `.review/` exclusion, and any one-off user criteria. For uncommitted changes, include staged, unstaged, and untracked files; for a base review, use the resolved comparison ref and merge base. From the second pass on, also give it the invariants of closed clusters, from every source, to re-check for bypasses, and the dismissed signatures with their evidence, asking it to report a dismissed item again whenever that evidence no longer holds for the current code. These add checks and never narrow the review.

Before accepting a result, verify what was actually reviewed. A failed invocation or an incomplete review is not a completed round.

### Waiting for the reviewer

These rules apply to every backend. Use its actual lifecycle interface rather than assuming another harness's tools or notifications.

- Before waiting, record the reviewer, session, or process identifier, the fingerprint, and the state (`running`, `completed`, `failed`, or `interrupted`) in the ledger. Keep one active reviewer per pass, and do not change the reviewed inputs while it runs.
- Silence is not failure. Wait timeouts, missing progress reports, and elapsed time never by themselves justify declaring a review stalled, interrupting it, launching a replacement, or switching backends. Use the harness's status interface to tell running work from completion or failure; if needed, read only that reviewer's identified session log, read-only, without guessing paths or scanning other sessions. Record observed activity separately from inferred state. If progress cannot be observed, say so and keep waiting. Use bounded waits, and update the user at least every 60 seconds without claiming progress you have not seen.
- Interrupt only for an explicit user stop, an invalidated target, a contract violation, or a confirmed execution failure that cannot proceed, and record the evidence. Confirm the original reviewer has stopped before replacing it. Falling back to another backend requires established unavailability or an unrecoverable failure, plus every applicable approval.
- After an interruption or compaction, recover the recorded reviewer and check its state before dispatching another. If it is still running on the same fingerprint, keep waiting. If its final report is available, verify coverage and fingerprint before accepting it. An interrupted or incomplete attempt consumes no round, and a resumed attempt is still the same pass. Start the next reviewer only after the current pass has ended.
- **Codex:** `wait_agent` may not forward a child's commentary or tool activity. Before interpreting an empty wait, check the delegated agent's status interface, or its identified local session log when accessible.
- **Claude Code:** a foreground Agent call returns only when the subagent finishes, so the call itself is the wait; record the returned agent ID and final state. If the reviewer runs in the background anyway, wait for its completion notification without changing the reviewed inputs or launching another reviewer.

## Ledger

Read [references/local-ledger.md](references/local-ledger.md) when starting or resuming. The ledger holds the loop's history across rounds, runs, and sources. Preserve it, exclude it from every review, and never stage or commit it.

## Start and resume

1. Read the applicable `AGENTS.md`, then inspect `git status`, the current branch, HEAD, and the complete target diff. Preserve unrelated changes.
2. Resolve the target and the ledger path. If the ledger exists, reconstruct open findings, earlier fixes, and historical clusters from it.
3. Partition a cross-cutting target by state owner or lifecycle coordinator. Every changed subsystem must pass its cluster gates before the verification pass. If the subsystems cannot be reasoned about independently, recommend splitting the change instead of spending rounds.
4. Record a content fingerprint for the exact reviewed inputs, excluding the ledger: HEAD, the merge base for `--base`, and hashes covering the reviewed diff and untracked files when applicable. A review result is valid only for its fingerprint.
5. After an interruption, compaction, or any uncertainty, reread the ledger and recompute the fingerprint, then resume from the earliest incomplete step. If existing changes cannot be attributed safely, stop instead of overwriting them. Repeating a read-only review is safe; repeating an uncertain edit is not.

## Triage before editing

Cluster every finding before editing anything, using [references/cluster-closure.md](references/cluster-closure.md). In short:

1. **Signature.** Record each finding's trigger, lifecycle phase, persistence sink, effect, and signature `<state owner> / <invariant> @ <enforcement boundary>`. State the invariant as a property the state must keep, not as the symptom or the cited line.
2. **Strong match, then weak.** The same invariant on the same state owner joins one cluster, with boundary `?` until the boundary shared by all members is found. A shared phase, state owner, sink, file, or symptom alone only proposes a merge; accept it only when one hypothesis and one fix boundary explain both.
3. **History.** Test every finding against every historical cluster and dismissal, by meaning rather than wording, whatever their source. A cluster match is a recurrence at any boundary. A dismissal match is re-dismissed only after re-triage from scratch against the current code, including any new argument from the reviewer.
4. **Split or merge.** Keep a cluster only when its hypothesis predicts every member and names one fix boundary; otherwise split it. Merge clusters only when one hypothesis predicts both and one fix boundary covers both.
5. **Predict siblings.** Search the target diff and call graph for unreported violations of each cluster's invariant; verify each one, then fix or dismiss it before closure.
6. **Order.** Fix upstream boundaries first, then by severity and recurrence; re-triage downstream clusters after each upstream fix.

For every cluster, trace one complete path from trigger through persistence and retry or cleanup, inspect every caller and sibling path that crosses the boundary, and write a root-cause hypothesis before choosing a fix. Enter stabilization mode on any trigger listed in the reference, including two current findings in one cluster, a historical match, or an accepted weak-signal merge. A singleton cluster that meets none of them takes the reference's fast path.

Before treating a cluster as fixed, it must pass the closure gate in the reference. Do not start another review while a known cluster is open.

## Each round

1. Recompute the fingerprint and run one fresh independent review. Record the fingerprint and every actionable finding before editing.
2. Reconcile the findings with the ledger's history and validate them against the intended behavior, all callers, sibling paths, tests, and the complete diff. Dismiss false positives with concrete evidence. Then triage them into clusters.
3. For every valid cluster, complete any required inventory, then fix the narrowest shared boundary that restores the invariant. Avoid per-caller guards unless the callers' contracts intentionally differ.
4. Add or update the smallest regression scenarios that satisfy the closure gate, without duplicating equivalent tests.
5. Apply the closure gate over the complete target diff and the relevant call graph. If it fails, investigate rather than stacking another patch or starting a review.
6. Run focused validation and the practical checks the repository requires, and record the commands and results in the ledger.
7. Reinspect the diff, recompute the fingerprint, and update cluster and subsystem dispositions. Record the round's convergence signals from the reference and act on them before the next pass.

Start the next pass only when every known cluster is closed or concretely dismissed. Reserve the last allowed pass for verification: run it only after every known cluster passes the closure gate and required validation passes. If it still returns actionable findings, stop at the limit without claiming convergence.

## Recurrence means root-cause mode

When a new finding belongs to an earlier cluster, reopen the cluster and mark the earlier fix incomplete. Compare the current code with the ledger's earlier enforcement boundary and work out why that boundary was narrower than the invariant. Do not repeat the same leaf-level fix under another caller or condition.

Trace the end-to-end lifecycle and the shared state or contract, replace repeated guards with the smallest shared fix, and remove local patches it makes redundant. Cover both the earlier and the current symptom. Do not run another review until the reopened cluster passes the closure gate. If the root fix needs a product or architecture decision outside the scope, leave the cluster open and ask the user rather than spending more rounds.

A cluster that recurs again after a root-cause-mode fix goes to reframe mode instead.

## Reframe when the approach keeps leaking

Root-cause mode moves a fix to a better boundary but keeps the approach. When a cluster recurs after that, the approach itself is producing the findings, and no boundary or extra round will converge it. The usual shapes: code that interprets open-ended input (free text, shell syntax, URLs, number formats), infers a fact from indirect signals instead of reading it, or promises a guarantee wider than anything it can check. Each patch closes one variant and the next pass finds another.

1. Stop patching the cluster. Name the premise the current approach depends on, and restate every member finding as one way that premise fails.
2. List at least three structurally different approaches, not variants of the current one. Check each of these moves: remove the need (change the data format or contract so nothing is interpreted, such as splitting a field instead of parsing it back); read the fact from its direct source instead of inferring it; observe the effect instead of predicting it; narrow the guarantee to what can be checked and document the rest; move the work to the layer that owns the information; drop the feature.
3. Gather evidence with every means in scope: the dependency's documentation and source, small experiments, replays of real data or logs, and measured coverage and false positives for each approach. The fixing agent is anchored on its own approach, so also give a fresh subagent the problem, the premise, and the findings without the patch history, and ask it for approaches.
4. Choose the approach under which the cluster's whole class of findings cannot occur, not the one with the smallest diff. Record what disappears by construction and what new limits it brings.
5. If the approach stays within the requested behavior and scope, save the current diff outside the repository, revert the cluster's patch stack, implement the approach, rebuild regression scenarios from every member finding, and mark the old cluster `superseded`. Tell the next reviewer the removed premise and ask it to look for any path that still relies on it. If the approach changes user-visible behavior, a recorded decision, or the scope, mark the cluster `blocked`, present the comparison with a recommendation, and ask the user.

Record the premise, the approaches, the evidence, and the decision in the cluster's ledger entry. Reframing runs no review pass and consumes no round.

## Ingest external findings

`--ingest` runs findings that another workflow recorded, usually PR review feedback from `follow-pr`, through the same clusters as local findings. A PR comment that repeats a bug fixed during the local loop then reopens that cluster instead of getting a second local patch.

1. Resolve the target and the ledger path as usual, creating the ledger if it does not exist. The caller has added each new external finding to the ledger with status `open`, a `Source:`, and no cluster.
2. Verify each one against the current code, then triage all of them against the full history, local and external, as in [Triage before editing](#triage-before-editing). A match with any earlier cluster is a recurrence and uses root-cause mode.
3. Fix, cover, close, and validate as in steps 3 to 6 of [Each round](#each-round), including the predicted siblings.
4. Give every external finding a final status and a disposition written for the reviewer who raised it:
   - `fixed`: what changed and at which boundary.
   - `dismissed`: invalid, obsolete, or already addressed, with the evidence.
   - `deferred`: out of scope or pre-existing, with a suggested follow-up. It does not expand the change.
   - `blocked`: needs a decision from the user; state the question.
5. Record the cycle and its convergence signals under Rounds, then stop and report every external finding's status and disposition.

Ingest runs no review pass, consumes no `--max` round, and never claims convergence: the PR's reviewers and CI verify the result after the caller pushes. It does not commit or push; the caller does.

## Stop conditions

Repair validation failures caused by the requested changes, and rerun the affected checks within the authorized scope. A failed check prevents convergence; it does not by itself end repair work.

Declare convergence only when all of these hold:

- a fresh independent review after the latest edit returns no actionable findings for the current fingerprint;
- every subsystem and cluster is closed, where superseded or dismissed counts as closed;
- required validation passes.

A re-reported dismissal that still holds after re-triage from scratch, including the reviewer's new argument, with no edit, is not actionable; list it in the final report. The ledger alone, an earlier clean review, or unchanged filenames are not proof.

When the round limit is reached with a cluster that recurred, run reframe steps 1 to 4 before stopping and report the recommended approach, rather than only offering more rounds; more rounds of the same approach rarely converge.

Stop without claiming convergence when the round limit is reached with findings, validation is blocked beyond an in-scope repair, the target changes unexpectedly, a fix needs a product decision outside the requested scope, or the convergence signals show the target is too wide; in that last case, recommend how to split it.

Report the rounds used, fixes, dismissals, reframes with their decision, remaining findings, validation results, target HEAD and worktree state, the ledger path, and the exact stop reason.
