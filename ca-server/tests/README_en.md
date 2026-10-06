**[English](README_en.md) | [中文](README.md)**

# Test Directory Notes

## Layering Conventions

- `unit/`: pure unit tests, verifying only the logic of this repository, without depending on a real database or real network services.
- `integration/`: verify the assembly of the `ca-server` runtime itself with the test database, certificate materials and internal dependencies; external `registry-server` interaction must be expressed through fake registry verifiers, stub transports or contract fixtures, rather than requiring real sibling processes to run at the same time.
- `e2e/`: verify the black-box behavior of a managed, temporary `ca-server` instance, covering self-contained scenarios of this service such as ACME, certificate CRUD, CRL, OCSP, and admin-only management capabilities.

## Boundary Notes

- `tests/` in this repository does not host real cross-service integration e2e.
- Real integration chains involving `ca-server` with `registry-server` and `discovery-server` are uniformly verified in `acps-cli/tests/e2e/`.
- If a test scenario depends on real sibling services, shared sibling repository configuration, or cross-repository topology collaboration, then it does not belong to the long-term boundary of the `integration` / `e2e` of this repository.

## Running Entry Points

- `just test unit`: pure unit layer.
- `just test integration`: this service + test database + fake registry peer.
- `just test e2e`: managed startup of a temporary `ca-server` test instance, and injects `TEST_E2E_BASE_URL`.
- `just test`: execute `unit -> integration -> e2e` in sequence.

## Prerequisite Conventions

- `just test bootstrap` is responsible for establishing the test environment, preparing the test database schema and the test certificate materials of this repository.
- `just test integration` and `just test e2e` will not implicitly execute bootstrap; if the test environment is not ready, they will fail directly at the entry point check stage.
- If you need to bypass `just test ...` and run `uv run pytest` directly, please first make sure `TEST_DATABASE_URL` is configured in `.env`, and run `just prep migrate test` as needed.
- The current goal of the black-box e2e of this repository is not to depend on sibling repository environment files; if real cross-service integration is required, please switch to `acps-cli`.

## skip and warning Conventions

- Missing real sibling services, missing manually prepared tokens/certificates/environment files should not be avoided long-term through `skip`; such prerequisites should gradually be moved into bootstrap, fixture or managed test instance preparation logic.
- `skip` should only be applied to future work, capabilities not yet implemented, or explicitly unsupported platform paths.
- warning should be regarded as an issue to be fixed, handled first in test code, fixtures and the dependency compatibility layer.

## Debugging Suggestions

- To debug the black-box e2e of this repository, you can prefer `just test e2e`, letting the test entry point automatically start a temporary instance.
- To debug real cross-service behavior, please switch to the `acps-cli` repository, and run according to the integration test instructions in its README.
