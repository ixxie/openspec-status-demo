# beacons Specification

## Purpose
TBD - created from change add-beacon-registry. Update Purpose.

## Requirements

### Requirement: Beacon registry

The system SHALL register beacons by hardware id before accepting their telemetry.

#### Scenario: Registered beacon reports
- **WHEN** a registered beacon submits a reading
- **THEN** the reading is attributed to that beacon

#### Scenario: Unknown hardware
- **WHEN** an unregistered beacon submits a reading
- **THEN** the system responds 404

### Requirement: Beacon liveness

The system SHALL mark a beacon offline when no heartbeat arrives within its reporting interval.

#### Scenario: Heartbeat lapses
- **WHEN** a beacon misses its heartbeat window
- **THEN** the beacon is marked offline and operators are notified

### Requirement: Batched telemetry upload

The system SHALL accept a batch of telemetry readings in a single request.

#### Scenario: Replay after outage
- **WHEN** a gateway uploads a batch of buffered readings
- **THEN** every reading is accepted or the batch is rejected atomically
