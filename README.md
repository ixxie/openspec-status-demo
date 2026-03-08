# openspec-status-demo

A beacon-telemetry service using OpenSpec with the default `lifecycle: archive`
workflow: finishing a change means `openspec archive`, which folds its deltas
into `specs/` and moves the folder to `changes/archive/YYYY-MM-DD-<name>/`.

This repo will later migrate to the proposed `lifecycle: status` mode — watch
the history.
