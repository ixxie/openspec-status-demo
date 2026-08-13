# openspec-status-demo

A working demo of a proposed OpenSpec experimental mode, **`lifecycle: status`**:
a change's lifecycle state is **data** (a `status` field in its `.openspec.yaml`),
not **directory position** (a move into `changes/archive/`). Nothing ever moves.

Implementation lives on [`ixxie/OpenSpec#lifecycle-status`](https://github.com/ixxie/OpenSpec/tree/lifecycle-status),
which this repo installs directly:

```jsonc
// package.json
"dependencies": { "@fission-ai/openspec": "github:ixxie/OpenSpec#lifecycle-status" }
```

## Why

Under the default mode, `openspec archive` does two unrelated jobs in one
command — a **state transition** (this change shipped) and a **text merge**
(fold its deltas into `specs/`) — and encoding the transition as a directory
move welds the merge to a single moment in the PR lifecycle. That moment is
awkward everywhere and nonexistent on GitLab (no approval event, and pushing
after approval resets approvals).

With state as data:

- **shipping is a one-line diff** that merges trivially and needs no special moment
- the merge is a standalone, **idempotent** `openspec sync`
- and the gate becomes a **tree-level predicate** — `shipped ⇒ folded` — that a
  pure file check evaluates identically at pre-commit, pre-push, and CI:

```sh
openspec sync --check   # exit 1 iff any shipped change has unfolded deltas
```

## What's in this repo

A tiny fictional beacon-telemetry service, with changes in every lifecycle state:

| Change | Status | Meaning |
|---|---|---|
| `add-user-auth` | `shipped` | deltas folded into `openspec/specs/auth/` |
| `beacon-batch-upload` | `proposed` | in flight — its deltas stay **out** of `specs/`, and the gate stays green |

`openspec/specs/` therefore describes **shipped reality only** — the invariant
the archive workflow was protecting, kept without any moves.

## Try it

```sh
npm install
npm run hooks        # git config core.hooksPath .githooks

npx openspec list                    # every change, with its lifecycle state
npx openspec list --status proposed  # the in-flight set
npx openspec sync --check            # the predicate — green on this tree

# the full ship flow, one atomic diff:
npx openspec ship beacon-batch-upload
git diff --stat                      # status flip + fold, one commit-able diff
git checkout .                       # undo the demo
```

## The gate, live

The same predicate runs in three places, in increasing authority:

1. **pre-commit** (`.githooks/pre-commit`) — refuses to author an incoherent commit
2. **pre-push** (`.githooks/pre-push`) — last local tier (hooks are advisory; `--no-verify` exists)
3. **CI** ([`gate.yml`](.github/workflows/gate.yml)) — authoritative, on every PR

See the demonstration PR, where a change is flipped to `shipped` **without**
folding — CI goes red — and the follow-up commit runs `openspec sync` — CI
goes green. The check is deterministic: no model, no network, no VCS history.

## What this deliberately does not solve

Two changes editing the same requirement still collide (upstream's
parallel-merge territory), and an edited delta cannot re-merge over an earlier
fold without base snapshots. The mode changes **when** the merge may run —
any time — not **how** it merges.
