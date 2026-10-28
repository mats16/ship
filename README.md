# ship

Agent skills that take a code change to an approved pull request.

| Skill | What it does |
|---|---|
| `ship` | Runs `simplify`, `review-loop`, `create-pr`, and `follow-pr` in that order, with a gate after the review loop. Invoked only by the user. |
| `simplify` | Deletes, reuses, and straightens the recent diff without changing observable behavior. |
| `review-loop` | Runs a bounded local review-fix-review loop with a fresh, independent reviewer each round, clustering findings by root cause until a clean review. When a cluster keeps recurring, it stops patching and compares structurally different approaches. No GitHub mutations. |
| `create-pr` | Branches, commits, and opens or updates the PR. |
| `follow-pr` | Follows an open PR through CI failures and review comments until it is approved. Review findings go through `review-loop`'s clusters, so they join the local review history. |

## Install

### Claude Code

```
/plugin marketplace add mats16/ship
/plugin install ship@ship
```

Then run `/ship:ship` on a change. The other skills are `/ship:simplify`, `/ship:review-loop`, `/ship:create-pr`, and `/ship:follow-pr`. Update later with `/plugin marketplace update ship`.

### Codex

```sh
codex plugin marketplace add mats16/ship
codex plugin add ship@ship
```

Then run `$ship:ship` on a change. Update later with `codex plugin marketplace upgrade ship`.

### Other harnesses that read `SKILL.md`

Clone the repository and link the skills into your skills directory, for example:

```sh
git clone https://github.com/mats16/ship ~/Projects/ship
for s in ship simplify review-loop create-pr follow-pr; do ln -s ~/Projects/ship/skills/$s ~/.agents/skills/$s; done
```

Then run `/ship` (or `$ship`). Installed this way, the skills use their bare names.

## Usage

```
/ship:ship [--ja] [--draft] [--no-follow] [--max <rounds>] [instructions]
```

- `--max` sets the number of review rounds for `review-loop` (default 5).
- `--ja` writes the PR in Japanese. `--draft` opens it as a draft. `--no-follow` stops after the PR is created instead of running `follow-pr`.
- Free-form instructions (base branch, issue to link, scope) go to `create-pr`.

`review-loop` keeps its working notes in `.review/<branch>.md` at the repository root, and `follow-pr` adds the PR's review findings to the same file. Neither commits it; add `.review/` to your global gitignore if you like.

## Requirements

- `git` and an authenticated `gh` CLI.
- In Claude Code, `review-loop` runs each review through the `/code-review` skill in a subagent on the same model and effort as the main agent.

## License

MIT
