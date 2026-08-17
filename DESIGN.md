# The design

A working demo of a proposed OpenSpec experimental mode, `lifecycle: status`.
This file explains what the mode is and why it's shaped this way. For the
narrated tour of this repo's history, see the [README](README.md).

## The problem: one command, two jobs

`openspec archive` does two unrelated things at once:

1. a **state transition** — declaring "this change is finished"
2. a **text merge** — folding the change's spec deltas into `openspec/specs/`

Bundled, they're convenient. The trouble is that the transition is encoded as a
*directory move* (`changes/x/` → `changes/archive/2026-08-17-x/`), and that move
drags the merge along with it to whatever moment the move happens. On a solo
repo that's fine. On a team with code review, every candidate moment is wrong
somewhere:

| When you archive | What goes wrong |
|---|---|
| During the PR | Review feedback invalidates the fold. There is no un-archive, and re-archiving isn't a no-op. |
| After merge | A bot has to commit to a protected branch, racing other merges. |
| At approval | GitLab has no approval event at all — `CI_MERGE_REQUEST_APPROVED` fires before the pipeline exists, and any push resets approvals. |

Upstream's own team-workflow docs present both conventions and advise "pick one
and be consistent." That's a choice between costs, not a solution.

## The move: state becomes data

Under `lifecycle: status`, a change records its own state:

```yaml
# openspec/changes/add-user-auth/.openspec.yaml
schema: spec-driven
created: 2026-03-15
status: shipped        # ← proposed | shipped
```

Nothing ever moves. There is no `archive/` directory. The state is one line that
merges trivially, needs no special moment, and is corrected by editing it.

Two states, not more. Each one earns its place by having a *machine
consequence*:

- **`shipped`** — "these deltas belong in `specs/`." This is what `sync` folds
  and what the gate checks.
- **`proposed`** — "this change holds a live claim on the requirements it
  touches." This is what overlap and drift tooling can reason over, without
  having to infer liveness from a directory path.

A state with no consequence would just be a comment. That's why there's no
`applied`: "implementation is done" is already recorded by `tasks.md`
checkboxes, and a second record of the same fact drifts from the first.

## The gate: a predicate, not a schedule

Once state is data, the merge becomes an ordinary command that can run whenever
you like:

```sh
openspec sync           # fold every shipped change's deltas into specs/
openspec sync --check   # verify, don't write; exit 1 if anything is unfolded
openspec ship <change>  # flip to shipped and fold, as one diff
```

And the thing CI enforces stops being a *timing condition* and becomes a
*predicate over the working tree*:

> **shipped ⇒ folded**

That distinction is the whole point. "Did archive run at the right moment?" is
unanswerable mid-PR — that's exactly when the invariant is *supposed* to be
violated. "Does this tree satisfy shipped ⇒ folded?" is answerable by anyone,
on any tree, at any time. So one command gates every altitude:

```sh
openspec sync --check   # .githooks/pre-commit · .githooks/pre-push · CI
```

No model. No network. No git history. Just files on disk.

### Folded-ness is decided by regeneration

A change counts as folded when **re-applying its delta to the current spec
produces byte-identical output**. Not a hash sidecar, not a lockfile, not a
timestamp comparison — the engine rebuilds the spec and compares bytes.

Two consequences worth naming:

- There is no bookkeeping state that can rot or be corrupted, because there is
  no bookkeeping.
- `--check` and the actual fold run the **same code**. The only difference is
  whether the rebuilt bytes get written. That's deliberate: upstream issue
  [#1112](https://github.com/Fission-AI/OpenSpec/issues/1112) exists because
  `validate` accepted deltas that `archive` then refused — a checker that
  reimplements the doer will always eventually disagree with it.

The cost, stated plainly: this is O(shipped history) per run, where archive's
implicit gate was O(active changes). Fine for young histories; a `--changed`
scope is the obvious refinement for large ones, and is deliberately *not* part
of the initial proposal.

## What this buys you

The bookkeeping stops being scheduled repo-wide and becomes a per-change choice:

| | `lifecycle: archive` | `lifecycle: status` |
|---|---|---|
| Where bookkeeping lands | end of PR, or post-merge main — fixed for everyone | any commit or PR — per change |
| The declaration itself | folder move + `specs/` rewrite | one line |
| Rides with the implementation commit | practically no | yes |
| Review iteration | un-archive → edit → re-archive | edit, re-run idempotent `sync` |
| Commits to protected main after merge | required by convention | never |
| Enforceable in CI | no — it's a timing condition | yes — it's a tree predicate |

## What it deliberately does *not* solve

Honesty about the edges matters more than a clean pitch:

- **Two open changes editing the same requirement** still collide. This changes
  *when* the merge may run, not *how* it merges. That's upstream's parallel-merge
  territory ([#1669](https://github.com/Fission-AI/OpenSpec/issues/1669)) — and
  a first-class `proposed` state is arguably a *help* there, since overlap
  detection currently has to infer which changes are live from where their
  folders sit.
- **Editing a delta after it was already folded** needs base snapshots to
  re-merge correctly. That window exists today; making fold-anytime normal means
  it sees more traffic. The gate detects the state and fails closed rather than
  silently corrupting `specs/`.
- **`ls` stops being the answer to "what's in flight."** `openspec list --status
  proposed` is. Once state is data, the filesystem is no longer the UI for
  state. This is the real cost of the design, and it's why the mode is opt-in.
- **Atomicity is a discipline, not a guarantee.** Flipping to `shipped` in one
  commit and folding in a later one leaves intermediate commits failing the
  predicate. `openspec ship` makes the atomic path the easy path, and the
  pre-commit hook rejects an incoherent commit — but `--no-verify` exists, and
  CI only ever sees the head tree.

## Scope: what's actually being proposed upstream

This repo demonstrates more than the proposal asks for. Keeping the two separate:

**Proposed now** — the mode itself: the `lifecycle` config flag, the `status`
field, `sync` / `sync --check` / `ship`, the `list` surface, and `archive`
refusing to run under status mode so the two models can never both claim a
change.

**Demonstrated here but deliberately *not* proposed** — the layout. This demo
shards change directories by creation date (`changes/2026/03/15-add-user-auth/`)
and includes a bidirectional `openspec migrate`. That was our answer to "if
nothing moves, how does `changes/` stay navigable?" — but upstream
[PR #1367](https://github.com/Fission-AI/OpenSpec/pull/1367) answers it better,
with user-chosen *domains* discovered by a leaf marker rather than a date
convention parsed out of regexes. Layout is a separable question, and that PR
should win it. The date sharding you see in this repo's paths is scaffolding for
the demo, not part of the pitch.
