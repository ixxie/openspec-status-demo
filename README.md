# openspec-status-demo

A working demo of a proposed OpenSpec experimental mode, **`lifecycle: status`** —
including a **live migration from the legacy workflow**. The history of this repo
is the demo: read it commit by commit.

> **Start with [DESIGN.md](DESIGN.md).** It explains what the mode is, why
> `archive` doing two jobs in one command is the problem, why the gate is a tree
> predicate (`shipped ⇒ folded`) rather than a schedule, what the design
> deliberately does *not* solve — and which parts of this demo are actually being
> proposed upstream versus which are scaffolding.

Implementation lives on [`ixxie/OpenSpec#lifecycle-sharding`](https://github.com/ixxie/OpenSpec/tree/lifecycle-sharding)
(stacked on [`lifecycle-status`](https://github.com/ixxie/OpenSpec/tree/lifecycle-status)),
which this repo installs directly:

```jsonc
// package.json
"dependencies": { "@fission-ai/openspec": "github:ixxie/OpenSpec#lifecycle-sharding" }
```

## Act I — five months of the legacy workflow (March–August)

The history begins as an ordinary `lifecycle: archive` project: three changes
proposed, implemented, and finished with real `openspec archive` runs —
`add-user-auth` (March), `add-beacon-registry` (May), `beacon-heartbeat` (July) —
each fold landing in `specs/` and each folder moving to
`changes/archive/YYYY-MM-DD-<name>/`. A fourth change, `batch-upload`, is still
in flight when Act II begins.

## Act II — one command migrates everything

```
$ npx openspec migrate
  ✓ add-user-auth        → openspec/changes/2026/03/15-add-user-auth  [shipped]
  ✓ add-beacon-registry  → openspec/changes/2026/05/02-add-beacon-registry  [shipped]
  ✓ beacon-heartbeat     → openspec/changes/2026/07/21-beacon-heartbeat  [shipped]
  … batch-upload         → openspec/changes/2026/08/01-batch-upload  [proposed]
```

- Archived changes become **`status: shipped`**, sharded by creation date —
  `changes/YYYY/MM/DD-<name>/`, the same information the archive folder name
  carried, with no `archive/` directory left. Nothing was deleted.
- The in-flight change becomes **`status: proposed`**, sharded by its
  `created:` date.
- `openspec sync --check` verifies every migrated shipped change by
  regeneration — the engine's folds re-apply byte-identically, so the gate is
  green immediately after migration.

State is now **data, not directory position**: a change's `.openspec.yaml`
carries `status: proposed | shipped`, and nothing ever moves again.

## Act III — the new workflow

- `openspec list` shows every change with its lifecycle state; `specs/`
  describes **shipped reality only**.
- `openspec ship <change>` declares shipped and folds, as **one atomic diff**.
- The gate is one tree-level predicate — **`shipped ⇒ folded`** — checked
  identically at pre-commit (`.githooks/`), pre-push, and CI
  ([`gate.yml`](.github/workflows/gate.yml)): deterministic, no model, no
  network, no VCS history.

See the demonstration PR: a change flipped to `shipped` **without** folding
(hooks bypassed with `--no-verify`) turns CI **red** with the remediation named;
one `openspec sync` commit turns it **green**. That two-commit timeline is the
argument: CI enforces at review time what the archive workflow could only hope
someone remembered at merge time.

## Try it

```sh
npm install
npm run hooks                        # git config core.hooksPath .githooks

npx openspec list                    # states across both migrated eras
npx openspec list --status shipped   # the folded history
npx openspec sync --check            # the predicate — green on this tree
npx openspec ship batch-upload       # flip + fold, one diff (then: git checkout .)

# the experiment is leaveable — reversal is a pure relayout, no spec text changes:
npx openspec migrate --to archive --dry-run   # every archive date maps back exactly
```

## What this deliberately does not solve

Two changes editing the same requirement still collide (upstream's
parallel-merge territory), and an edited delta cannot re-merge over an earlier
fold without base snapshots — migrated history that was superseded in place
would surface the same way. The mode changes **when** the merge may run — any
time — not **how** it merges.

[DESIGN.md](DESIGN.md#what-it-deliberately-does-not-solve) covers the full list,
including the honest costs: `ls` stops answering "what's in flight", and
atomicity is a discipline rather than a guarantee.

## Demo scaffolding vs. the actual proposal

This repo shows more than is being proposed. The **mode** — the `lifecycle` flag,
the `status` field, `sync` / `sync --check` / `ship`, and `archive` refusing under
status mode — is the proposal. The **layout** shown here (date-sharded
`changes/YYYY/MM/DD-<name>/` paths and the bidirectional `openspec migrate`) is
not: upstream [PR #1367](https://github.com/Fission-AI/OpenSpec/pull/1367) answers
the layout question better, with user-chosen domains found by a leaf marker rather
than dates parsed out of regexes. See
[DESIGN.md](DESIGN.md#scope-whats-actually-being-proposed-upstream).
