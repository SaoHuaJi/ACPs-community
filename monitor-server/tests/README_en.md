**[English](README_en.md) | [中文](README.md)**

# Test Directory Notes

## Test Layers

| Directory | Type | Characteristics |
|------|------|------|
| `unit/` | Unit tests | Mock external dependencies (DB, Kafka), extremely fast to run, no infrastructure required |
| `integration/` | Integration tests | Use real PostgreSQL (`agent_monitor_test`), transaction rollback isolation |
| `e2e/` | End-to-end tests | Verified through the HTTP Query API, require real DB + Kafka |

## How to Run

```bash
# Initialize the test environment (first time)
just test bootstrap

# Run all tests
just test all

# Run only unit tests (no infrastructure required)
just test unit

# Run only integration tests
just test integration

# Run only end-to-end tests
just test e2e

# Run only the Keycloak OIDC black-box integration tests
just test e2e -- tests/e2e/test_oidc_keycloak_flow.py

# Coverage statistics
just test coverage
```

## Prerequisites

- Integration tests and E2E tests require the PostgreSQL and Kafka provided by dev-infra:
  ```bash
  just infra up postgres kafka
  just infra wait postgres kafka
  ```
- `just test e2e -- tests/e2e/test_oidc_keycloak_flow.py` will additionally use the OIDC profile; the command will automatically execute `just infra up keycloak && just infra wait keycloak`.
- Database migration requires executing `just prep migrate test`.

## Environment Variables

| Variable | Description | Default Value |
|------|------|--------|
| `TEST_DATABASE_URL` | Test database connection string | `postgresql+asyncpg://monitor:monitor@localhost:5432/agent_monitor_test` |
| `APP_ENV` | Application environment (conftest automatically sets it to testing) | `testing` |
