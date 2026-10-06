**[English](README_en.md) | [中文](README.md)**

# Test Directory Notes

## Layering Conventions

- `unit/`: pure unit tests, without depending on external databases or network services.
- `integration/`: verify the integration behavior of the `discovery-server` runtime itself with the test database and local dependency assembly; external DSP / Registry interaction should preferably be expressed through fake upstreams, ASGI fake apps, stub transports or contract fixtures, rather than depending on real sibling processes.
- `e2e/`: verify the black-box behavior of a managed, temporary `discovery-server` instance, covering self-contained scenarios of this service.

## Boundary Notes

- `tests/` in this repository does not host real cross-service integration e2e.
- Integration workflows involving `discovery-server` with the real `registry-server` and `ca-server` should be uniformly verified in `acps-cli/tests/e2e/`.
- If a test requires sibling repository environment files, real sibling services, cross-repository forwarder topologies or shared external assets, then it does not belong to the long-term boundary of the `integration` / `e2e` of this repository.
- Legacy cross-boundary scenarios that still exist should be gradually migrated to `acps-cli` according to plan, rather than continuing to be expanded in this repository.

## Current e2e Entry Point

The current `just test e2e` randomly allocates a local port, manages the startup of a temporary test instance, waits for `/acps-adp-v2/health`, and injects `TEST_E2E_BASE_URL` before executing the black-box tests; this path depends on the shared PostgreSQL test database with pgvector, but no longer depends on any sibling repository environment files.

The current `just test integration`, `just test e2e` and `just test all` will all automatically rebuild the required test sample data before their respective suites start; after completing `just test bootstrap`, there is no need to manually execute `just prep seed test` or `just prep reseed test` in order to run the standard test entry points.

If `DISCOVERY_TEST_MODE` and `DISCOVERY_E2E_MODE` are not explicitly set, the standard `integration` / `e2e` / `all` entry points will by default execute both CPU and GPU modes in sequence; if you want to run only a single mode, you can override the corresponding environment variables before the command.

`just test bootstrap` (as well as `just test check`) will forcibly ensure that the test environment has the `gpu` extra installed (`uv sync --extra gpu`). This is a completely different scenario from "gracefully degrading and skipping when the test-mode GPU branch model files are missing", do not confuse the two:

- **gpu extra not installed** (`torch`/`FlagEmbedding` packages themselves missing): this is an environment preparation issue, `just test bootstrap` should have already filled it in before the tests run; if you still encounter it, it means the local environment bypassed bootstrap, and you should execute `uv sync --extra gpu` to fill it in, rather than letting the tests skip.
- **Local embedding model path/device missing** (`EMBEDDING_MODEL_PATH`, `EMBEDDING_DEVICES` not configured): this is a deliberate design trade-off that the test mode does not depend on real large-model files; the semantic matcher will gracefully degrade and skip startup, and black-box cases continue to verify mode switching, health checks and non-semantic paths, and this kind of degradation is expected behavior.

If you need a fixed port for debugging, you can explicitly set `DISCOVERY_E2E_PORT`; when it is not set, a random free port is used by default, consistent with the black-box e2e workflows of `registry-server` and `ca-server`.

The large-model-related variables required for e2e startup are resolved in the following order:

1. The explicitly set `DISCOVERY_E2E_*` environment variables.
2. The corresponding runtime variables in the `.env` of the current repository root.
3. Minimal placeholder values used only for service startup and basic liveness tests.

If subsequent e2e cases need to actually call embedding or the Discovery LLM, real test configuration should preferably be provided through `DISCOVERY_E2E_*` or the current repository `.env`, rather than depending on external sibling repository assets.

The test helper now prefers to read `TEST_E2E_BASE_URL`, and is compatible with `DISCOVERY_E2E_BASE_URL` during the transition period.

## skip and warning Conventions

- Missing sibling services, missing external environment files, or missing manually prepared forwarder topologies should not be avoided long-term through `skip`; such prerequisites should be converged into bootstrap, fixture or managed instance logic.
- `skip` should only be applied to future work, capabilities not yet implemented, or explicitly unsupported platform paths.
- warning should be regarded as an issue to be fixed, closed first in test code, fixtures and the dependency compatibility layer.

## Running Entry Points

- `just test unit`
- `just test integration`
- `just test e2e`
- `just test coverage`
- `just test`
- `just test all`

The current `just test` / `just test all` can already pass through the three test entry layers `unit`, `integration` and `e2e` in sequence; among them, `integration` and `e2e` by default run once each in both CPU and GPU modes, and the standard entry points run the corresponding suites by directory, no longer outputting `deselected` from other test layers.
