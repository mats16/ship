# Clustering, Stabilization, and Cluster Closure

## Clustering algorithm

Run these steps over all current findings plus every historical cluster and dismissal in the ledger. Do it before any edit, and again for downstream clusters after each upstream fix.

1. **Normalize.** For each finding, record trigger, lifecycle phase, persistence sink, effect, and a signature `<state owner> / <invariant> @ <enforcement boundary>`.
   - The state owner is the module, table, store, or coordinator that owns the state, not the file the reviewer cited.
   - State the invariant as a property the state must keep ("a job row is published only for a live, same-workspace session"), not as the symptom ("crash when session is deleted").
   - Choose the level of generality that one fix can enforce. Generalize until sibling violations fall under the same invariant, and stop before it covers state the fix boundary cannot see.
   - The enforcement boundary is the narrowest function, transaction, or constraint every violating path crosses. If unknown, write `?` and resolve it while tracing the path; a cluster cannot be fixed while its boundary is `?`.
2. **Strong match.** Two findings join one cluster when they have the same invariant on the same state owner, whatever boundaries they were reported at. The cluster's boundary is the one that dominates every member's, meaning it sits on every violating path, so one fix there covers all; keep it `?` until found. If tracing shows no single boundary on every violating path, split the members by boundary into sibling clusters that keep the invariant and name each other as siblings. Log it; step 2 does not undo that split. A later match on that invariant reopens the sibling whose boundary its path crosses, or opens a new sibling when it crosses none; either way it counts as a recurrence and a missed prediction, not a new cluster.
3. **Weak signals.** A shared phase, state owner, persistence sink, file, symbol, or symptom alone does not merge findings. It proposes a merge. Accept it only when one root-cause hypothesis predicts both findings and one fix boundary covers both. Otherwise keep them separate and note the rejected merge, so later rounds do not re-litigate it.
4. **History match.** Compare each finding with every historical cluster, including closed ones, and with every dismissal, using the step 2 test on meaning, not on the wording of the recorded invariant.
   - A cluster match is a recurrence at any boundary: a recorded boundary that differs only shows where the earlier fix was too narrow. Reopen the cluster (for a split invariant, the sibling chosen in step 2), re-examine that boundary, and use root-cause mode.
   - A match with a dismissal is re-triaged from scratch against the current code, including any new argument from the reviewer. Re-dismiss it only when the dismissal still holds; citing the earlier evidence alone is not enough.
5. **Split and merge.** A cluster is valid only when its hypothesis predicts every member and names exactly one fix boundary. If one member needs a different boundary, split it out. Merge two clusters only when one hypothesis predicts both and one fix boundary covers both, the same test as step 3. Log every merge, split, and rejected merge with its reason, and do not revisit a logged decision unless a member or boundary changed.
6. **Predict siblings.** For each cluster, search the target diff and call graph for code that can violate the invariant but was not reported: other callers of the boundary, paths that bypass it, sibling sinks, and the symmetric operation (create/delete, attach/detach, start/cancel). Add each hit as a `predicted` finding (`S<n>`) in the cluster. Verify each one like a reviewer finding, then fix or dismiss it with evidence before the cluster closes; this is what keeps the next pass from rediscovering them one at a time.
7. **Order.** Cluster A precedes B when A's boundary is upstream of B's or A's fix changes B's trigger. Fix in that order, then by severity, then by recurrence count. After each upstream fix, re-run steps 1-6 for downstream clusters. Close one that dissolves as superseded only when its original trigger passes as a regression and the predict step finds no path that bypasses the upstream boundary. Move its findings into the upstream cluster, whose closure gate then covers them. Superseded counts as closed.

## Fast path

A singleton cluster takes the fast path when its invariant is local: no stabilization trigger below applies, no shared or durable state, no asynchronous work, no identifier or ownership change, and the predict step found no siblings. Fix it, add the trigger regression, and record evidence for closure conditions 1, 4, and 5 below (for 5, the trigger alone, since there is no sibling path); mark the others not applicable with that reason. If any of those properties appears later, move the cluster to stabilization.

## Convergence signals

After each round, record in the ledger the counts of new clusters with at least one finding that survived validation, recurrences, re-dismissals, and missed predictions (new members of an existing cluster that the predict step did not find). Act on them before the next pass:

- **Recurrence after a closed cluster.** The invariant or boundary was too narrow. Restate the invariant one level more general and redo the predict step for that cluster before it can close again.
- **A missed prediction.** Record which search missed it and rerun the widened search for every cluster, closed ones included. Reopen any cluster that gains a predicted finding.
- **The same signature reported again after a dismissal.** Independent reviewers disagree with it. When the step 4 re-triage still holds, make its evidence more concrete in the next reviewer prompt.
- **Two consecutive rounds after round 1 in which every finding that survives validation opens a new cluster, each round opens at least two such clusters, and the second opens no fewer than the first.** Findings are not converging and the target is too wide for the remaining rounds. Stop and recommend how to split the change rather than spending them.

## Stabilization mode

Stabilization mode blocks another review pass until the affected cluster closes. Enter it when:

- a finding matches a historical cluster under the history-match step;
- two current findings share a cluster, or a weak-signal merge was accepted;
- multiple fixes touch the same lifecycle coordinator or durable state;
- asynchronous work and its detach, restore, retry, cancellation, deletion, or shutdown behavior both changed; or
- an earlier fix protected one caller while sibling callers can reach the same boundary.

Before editing a stabilizing cluster, add this inventory to the ledger:

| Entry point | Async result or operation | Persistence sink | Publish-time guard | Retry/cancellation | Regression scenario | Classification |
| --- | --- | --- | --- | --- | --- | --- |

Include every `predicted` finding as a row. Classify every row as protected by the shared fix, intentionally exempt with a concrete reason, or requiring work. Do not leave unknown rows.

## Closure gate

A cluster is closed only when all applicable conditions hold. Record pass evidence or a concrete not-applicable reason for each condition:

1. The root-cause hypothesis explains every historical, current, and predicted finding in the cluster.
2. Every entry point, caller, sibling path, and persistence sink crossing the invariant is classified.
3. For asynchronous or persisted results, the invariant is revalidated at the transaction or narrowest boundary that publishes the result, not only before the work begins.
4. The invariant is enforced at the narrowest boundary shared by all affected paths; remaining per-caller guards have intentionally different contracts.
5. Regression coverage includes the reported trigger, at least one sibling path, and stale-result, detach, cancellation, retry, or recovery behavior when relevant.
6. A repository search confirms that no unclassified bypass remains and the fix creates no new lifecycle gap.

If any condition is unmet, keep the cluster in stabilization mode. A reported test passing is not sufficient to close it.

When the target changes identifiers, ownership, revisions, foreign keys, or graph relationships, also complete a trust-boundary matrix covering malformed and duplicate IDs, missing and cross-scope parents, self-reference, supported depth, direct and indirect cycles, deletion with dependents, stale revisions, and concurrent create or update. Mark irrelevant cells explicitly. Prefer database constraints for invariants the database can express while preserving structured errors at the application boundary.
