# auth Specification

## Purpose
TBD - created from change add-user-auth. Update Purpose.

## Requirements

### Requirement: Operator authentication

The system SHALL authenticate beacon operators with bearer tokens before accepting telemetry.

#### Scenario: Valid token
- **WHEN** an operator presents a valid bearer token
- **THEN** the request is accepted

#### Scenario: Missing token
- **WHEN** a request carries no token
- **THEN** the system responds 401
