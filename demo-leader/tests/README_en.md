**[English](README_en.md) | [中文](README.md)**

# Leader Agent Platform Test Suite

## Test Categories

### Unit Tests (`tests/unit/`)

Unit tests use Mock objects to simulate external dependencies and do not require a real LLM service. They run quickly and are suitable for frequent execution during development.

**How to run:**

```bash
cd demo-leader

# Run unit tests (serial execution by default)
just test unit
```

> **Serial by default**: Unit tests use serial execution mode (`-n 0`) by default; no additional arguments are needed.

### Integration Tests (`tests/integration/`)

The basic principle of this project's integration tests is not to use Mocks: they use real LLM API calls and a real Partner service, verifying through both API responses and internal state.

> **Exception**: The Fallback test classes of some tests use Mocks to simulate service failures in order to test error-degradation logic.

**How to run:**

```bash
# Run from the project root
cd demo-leader

# Prepare the local test environment
just test bootstrap

# In a separate terminal, start demo-partner first
cd ../demo-partner && just dev start

# Run integration tests (file-level concurrency with 2 workers by default)
just test integration

# Specify the number of workers
just test integration -- -n 4 -v

# Run serially (parallelism disabled, for debugging)
just test integration -- -n 0 -v
```

> **Parallel by default**: Integration tests use `-n 2 --dist=loadfile` by default. This preserves the order of test cases within the same test file while allowing different test files to run concurrently. Use `-n 0` to switch to serial mode for debugging; the default number of workers can also be adjusted through the `JUST_TEST_LOADFILE_WORKERS` environment variable.

**Test groups:**

| Test file                        | Feature  | Description                        |
| -------------------------------- | -------- | ---------------------------------- |
| test_intent_analyzer.py          | LLM-1    | Intent analysis and scenario recognition |
| test_planning_flow.py            | LLM-2    | Full planning flow                 |
| test_clarification_flow.py       | LLM-3    | Clarification-and-merge flow (includes Fallback tests) |
| test_input_router_flow.py        | LLM-4    | Incremental update (InputRouter) routing flow |
| test_completion_gate_flow.py     | LLM-5    | Full CompletionGate flow           |
| test_aggregator_flow.py          | LLM-6    | Result aggregation flow (includes Fallback tests) |
| test_history_compression_flow.py | LLM-7    | History compression flow           |
| test_task_new_flow.py            | Flow     | Full TASK_NEW processing flow      |
| test_chitchat_flow.py            | Flow     | CHIT_CHAT chitchat flow            |
| test_multi_turn.py               | Flow     | Multi-turn conversation flow       |
| test_scenario_switch.py          | Flow     | Scenario switching flow            |
| test_idempotency_flow.py         | Validation | Idempotency / mode / activeTaskId validation |
| test_async_execution_flow.py     | Async    | Asynchronous execution mode (/submit + /result) |
| test_session_creation.py         | Session  | Session creation                   |
| test_session_ttl.py              | Session  | Session TTL expiration (patch-based time control) |

### End-to-End Tests (`tests/e2e/`)

End-to-end tests (E2E) are fully black-box tests: all external services run for real, business correctness is verified only through HTTP API requests and responses, and no internal data or modules are used for assertions.

**How to run:**

```bash
# Run from the project root
cd demo-leader

# Prepare the local test environment
just test bootstrap

# Run E2E tests (serial execution by default)
just test e2e

# Run the Keycloak OIDC black-box integration
just test e2e -- tests/e2e/test_oidc_keycloak_flow.py

# Run serially (parallelism disabled, for debugging)
just test e2e -- -n 0 -v
```

> **Serial by default**: E2E tests remain serial by default. In real-environment benchmarks, default concurrency introduces a read-timeout regression in `test_edge_cases.py::TestErrorHandling::test_empty_query`, so only explicit opt-in concurrency is kept here: for experiments you can pass `-n 2 --dist=loadfile` manually, and to fully disable xdist you can continue to pass `-n 0`.
>
> `tests/e2e/conftest.py` starts a temporary runtime under management, so there is **no need** to start application processes manually. Regular E2E brings up a temporary `demo-leader` HTTPS instance and a `demo-partner` runtime; if only the OIDC file is selected, `just test e2e -- tests/e2e/test_oidc_keycloak_flow.py` uses the lightweight `leader` runtime and automatically brings up `Keycloak` from dev-infra.

**Test groups:**

| Test file            | Description                                    |
| -------------------- | ---------------------------------------------- |
| test_api_contract.py | API contract tests (request/response format validation) |
| test_edge_cases.py   | Edge case tests (session management, error handling, idempotency) |
| test_oidc_keycloak_flow.py | Keycloak OIDC black-box integration (issuer / roles / session ownership) |
| test_user_journey.py | **User journey tests** (12 full rounds of interaction in the same session) |

**Focus: User Journey Test (`test_user_journey.py`)**

The user journey test simulates a real user performing 12 full rounds of interaction in **the same session**, verifying:

- Context accumulation and understanding
- History compression triggering
- Scenario switching (chitchat ↔ task)
- Incremental update handling
- Exception recovery
- State consistency across long chains

It contains 6 Phases (all tests share the same session):

- `TestPhase1_Opening`: Opening greeting and capability discovery (Turn 1-2)
- `TestPhase2_InitiateTask`: Initiating a travel planning task (Turn 3)
- `TestPhase3_SupplementInfo`: Supplementing information and change requests (Turn 4-6)
- `TestPhase4_IntermittentChitchat`: Intermittent chitchat in between (Turn 7-8)
- `TestPhase5_ErrorRecovery`: Invalid input and recovery (Turn 9-10)
- `TestPhase6_Conclusion`: Summary and verification (Turn 11-12)

## Test Command Summary

| Test type | Command                 | Default mode           | External dependencies |
| -------- | ----------------------- | ---------------------- | -------------- |
| Unit tests | `just test unit`        | Serial                 | None           |
| Integration tests | `just test integration` | `-n 2 --dist=loadfile` | LLM + Partner  |
| E2E tests | `just test e2e`         | Serial                 | Temporary Leader + temporary Partner + RabbitMQ |
| OIDC black-box | `just test e2e -- tests/e2e/test_oidc_keycloak_flow.py` | Serial | Temporary Leader + Keycloak (submit-related assertions still require an available LLM) |

> **Note**: The current default execution mode is controlled by the `Justfile` at the repository root, not by standalone `pytest.ini` files under each test directory. `unit` / `api` / `e2e` are serial by default, while `integration` uses `-n 2 --dist=loadfile` by default.

## Other Tests

```bash
# Run all tests (using each directory's default mode)
just test all

# With a coverage report
uv run pytest tests/ -v --cov=leader/assistant --cov-report=term-missing -n 0
```

## Notes

1. Integration tests require a valid LLM configuration (`config.toml`)
2. Integration tests still depend on an available Partner/LLM environment; regular E2E starts temporary `demo-leader` and `demo-partner` runtimes under test-fixture management
3. `just test e2e -- tests/e2e/test_oidc_keycloak_flow.py` uses the lightweight `leader` runtime and automatically runs `just infra up keycloak && just infra wait keycloak`
4. The submit / session permission assertions in the OIDC black-box tests depend on a real LLM; if the LLM is unavailable, the cases are skipped by design
5. When test runs take a long time, you can use the `-x` argument to stop at the first failure
6. Use `--tb=short` to simplify error output
