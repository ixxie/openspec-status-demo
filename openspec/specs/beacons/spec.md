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
