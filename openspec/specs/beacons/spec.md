# beacons Specification

## Purpose
TBD - created from change beacon-batch-upload. Update Purpose.

## Requirements

### Requirement: Batched telemetry upload

The system SHALL accept a batch of telemetry readings in a single request.

#### Scenario: Replay after outage
- **WHEN** a gateway uploads a batch of buffered readings
- **THEN** every reading is accepted or the batch is rejected atomically
