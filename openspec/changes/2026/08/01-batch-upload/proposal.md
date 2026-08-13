# Proposal

## Why
Field gateways buffer readings offline and currently replay them one request at a time; a batch endpoint cuts reconnect storms after outages.

## What Changes
- Accept batched telemetry uploads.
