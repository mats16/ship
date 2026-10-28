# Local Ledger

Keep history in one Markdown file at `.review/<branch>.md` under the repository root, replacing `/` in the branch name with `__`; use `detached-<short-head>` when detached. One ledger holds every source of findings for the branch: local review passes, predicted siblings, and external findings such as PR review feedback recorded by `follow-pr`. This file is working state, not product code:

- Never stage or commit it.
- Exclude `.review/` from every review target.
- Preserve it after convergence so a later run on the branch can detect recurrence.
- Do not add a database, helper script, or `.gitignore` change just for the ledger.
- When the work moves to a branch with another name, move the ledger to the new branch's path so its history follows the work.

Create the ledger with this minimum shape and extend existing entries rather than rewriting history:

```markdown
# Local review ledger
Harness / review backend: <harness and selected reviewer or command>
Reviewer model / effort: <main agent's model ID> / <effort level>
Target: base <ref> | uncommitted
Current fingerprint: <exact reviewed input fingerprint>

## Subsystems
- <state owner or lifecycle coordinator>: open | closed | intentionally exempt

## Clusters
### C1 — <state owner> / <invariant> @ <enforcement boundary>
- Root-cause hypothesis: ...
- Status: open | stabilizing | fast path | closed | dismissed | blocked | superseded
- Siblings: C<n> (same invariant, different boundary) | none
- Findings: R1-F1, S1 (predicted), ...
- Order: after C<n> | none; severity; recurrences <count>
- Merge/split log: <round> merged C<n> | split R1-F2 out | rejected merge with C<n> — reason
- Reframe: <round>; premise; approaches compared with evidence; decision | none
- Closure evidence: callers, persistence boundaries, regression scenarios, repository search

## Findings
- R1-F1 [P2] path:line — title  (predicted findings use S<n>; external findings use P<cycle>-F<n>)
  - Source: local pass R1 | predicted | PR #<n> <review | thread | comment> <url> by <login>
  - Scenario/effect: ...
  - Signature: <state owner> / <invariant> @ <enforcement boundary>
  - Trigger / phase / persistence sink: ...
  - Status: open | predicted | fixed | dismissed | deferred | blocked | superseded
  - Disposition: evidence, fix or dismissal reason, or suggested follow-up
  - Validation: command and result
  - Reply (external only): reply URL; thread resolved yes | no | left to reviewer

## Rounds
- Round 1 — fingerprint <value>; result <finding IDs or no findings>; new clusters <n>, recurrences <n>, re-dismissals <n>, missed predictions <n>
- PR cycle P1 — target base <ref>, head <sha>; ingested <finding IDs>; new clusters <n>, recurrences <n>, re-dismissals <n>, missed predictions <n>
```

Use stable finding and cluster IDs. Keep prior findings, fixes, dispositions, closure evidence, and validation even when later rounds no longer report them. Record the enforcement boundary changed by each fix so a recurrence can be compared with the earlier fix, and keep dismissed findings with their signature and evidence so later rounds can match them and re-check that evidence; raw reviewer transcripts are unnecessary.

