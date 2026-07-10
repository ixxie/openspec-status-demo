# Beacons - Changes

## ADDED Requirements

### Requirement: Beacon liveness

The system SHALL mark a beacon offline when no heartbeat arrives within its reporting interval.

#### Scenario: Heartbeat lapses
- **WHEN** a beacon misses its heartbeat window
- **THEN** the beacon is marked offline and operators are notified
