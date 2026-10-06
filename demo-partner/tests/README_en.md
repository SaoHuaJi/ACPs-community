**[English](README_en.md) | [中文](README.md)**

# Partner Agent Test Suite

## Test Categories

This project contains three categories of tests, covering the complete test pyramid from unit to end-to-end.

### Unit Tests (`tests/unit/`)

Unit tests use Mock objects to simulate the LLM service and do not depend on real external services. They run fast and are suitable for frequent execution during development.

**Test Coverage:**

| Test File                  | Corresponding Function | Description                                 |
| -------------------------- | ---------------------- | ------------------------------------------- |
| test_generic_runner.py     | GenericRunner core flow | Decision/Analysis/Production stages, concurrency control |
| test_task_state_machine.py | Task state machine     | State transitions, terminal state handling, message history |
| test_config_loading.py     | Configuration loading  | ACS/Config/Prompts configuration loading    |

**How to Run:**

```bash
# Run from the project root directory (demo-partner/)
uv run pytest tests/unit/ -v

# Run a specific test file
uv run pytest tests/unit/test_generic_runner.py -v

# Run a specific test class
python -m pytest tests/unit/test_generic_runner.py::TestDecisionStage -v

# Run a specific test method
python -m pytest tests/unit/test_generic_runner.py::TestDecisionStage::test_decision_accept -v
```

> **Serial by default**: Unit tests use serial execution mode (`-n 0`) by default, and no additional parameters need to be specified.

### Integration Tests (`tests/integration/`)

Integration tests use the real LLM service and call the Partner API through the FastAPI TestClient. They verify the correctness of the components working together.

> **Principle**: Do not use Mock (except for Fallback tests); use the real LLM API and verify through both API responses and internal state.

**Test Coverage:**

| Test File               | Corresponding Function | Description                  |
| ----------------------- | ---------------------- | ---------------------------- |
| test_api_basics.py      | API basic functionality | Health check, RPC endpoints, request validation |
| test_beijing_food.py    | Beijing food Agent     | Intent recognition, requirement analysis, content generation |
| test_beijing_rural.py   | Beijing rural Agent    | Intent recognition, requirement analysis, content generation |
| test_beijing_urban.py   | Beijing urban Agent    | Intent recognition, requirement analysis, content generation |
| test_china_hotel.py     | Nationwide hotel Agent | Intent recognition, requirement analysis, content generation |
| test_china_transport.py | Nationwide transport Agent | Intent recognition, requirement analysis, content generation |

**How to Run:**

```bash
# Run from the project root directory (demo-partner/)
uv run pytest tests/integration/ -v

# Specify the number of workers (4-8 recommended)
uv run pytest tests/integration/ -n 4 -v

# Run serially (disable parallelism, for debugging)
uv run pytest tests/integration/ -n 0 -v

# Run the tests for a specific Agent
uv run pytest tests/integration/test_china_transport.py -v

# Run a specific test class
uv run pytest tests/integration/test_china_transport.py::TestChinaTransportDecision -v
```

> **Parallel by default**: Integration tests use parallel execution mode (`-n auto`) by default, which can significantly shorten test time. Use `-n 0` to switch to serial mode for debugging.

### End-to-End Tests (`tests/e2e/`)

End-to-end tests (E2E) are completely black-box tests: they call a running Partner service through real HTTP requests and verify business correctness only through API responses.

**Key Characteristics:**

- Does not use TestClient; uses real HTTP requests (httpx)
- Requires the Partner service to actually be running (each Agent has its own port, such as 9021-9025)
- Performs multiple rounds of operations under the same task to verify the consistency of state transitions
- Must **execute serially**; cannot run in parallel

**Test Coverage:**

| Test File                     | Description                                  |
| ----------------------------- | -------------------------------------------- |
| test_state_machine_journey.py | **State machine journey tests** (multiple complete rounds of interaction under the same task) |

**Focus: State Machine Journey Tests (`test_state_machine_journey.py`)**

The state machine journey tests perform multiple rounds of operations under **the same task** to verify the complete state transition flow:

```
Journey 1: Incomplete request → AwaitingInput → Supplementary information → AwaitingCompletion → Completed
  START("Book a train ticket for me")
    → accepted → working → awaiting-input
  CONTINUE("Beijing to Shanghai, January 30")
    → working → awaiting-completion
  COMPLETE
    → completed

Journey 2: Complete request → Direct completion
  START("Look up tomorrow's high-speed rail from Beijing to Shanghai")
    → accepted → working → awaiting-completion
  COMPLETE
    → completed

Journey 3: Out of scope → Rejected
  START("Recommend great restaurants in Beijing")
    → accepted → rejected

Journey 4: Cancel midway
  START → awaiting-input → CANCEL → canceled
```

**How to Run:**

```bash
# Run from the project root directory (demo-partner/)
# Start the Partner service first
just dev start

# Run the E2E tests (serial execution by default)
uv run pytest tests/e2e/ -v -s

# Run a specific test class
uv run pytest tests/e2e/test_state_machine_journey.py::TestStateMachineJourney_ChinaTransport -v -s

# Run the state consistency tests (verify all Agents)
uv run pytest tests/e2e/test_state_machine_journey.py::TestStateTransitionConsistency -v -s
```

> **Serial by default**: E2E tests use serial execution mode (`-n 0`) by default, because the state machine tests need to perform multiple rounds of operations under the same task and execution order must be guaranteed.

## Quick Start

### Installing Dependencies

```bash
# Install dependencies (execute in the project root directory demo-partner/)
uv sync
```

### Running All Tests

```bash
# Run from the project root directory (demo-partner/)

# 1. Run unit tests (serial by default, no external services required)
uv run pytest tests/unit/ -v

# 2. Run integration tests (parallel by default, requires LLM API configuration)
uv run pytest tests/integration/ -v

# 3. Run E2E tests (serial by default, requires the Partner service to be started)
just dev start
uv run pytest tests/e2e/ -v -s
```

### With Coverage Report

```bash
# Run all tests and generate a coverage report
uv run pytest tests/ --cov=partners --cov-report=html -v

# View the coverage report
open htmlcov/index.html
```

## Test Command Summary

| Test Type | Command                            | Default Mode | External Dependency |
| --------- | ---------------------------------- | ------------ | ------------------- |
| Unit tests | `uv run pytest tests/unit/ -v`    | Serial       | None                |
| Integration tests | `uv run pytest tests/integration/` | **Parallel** | LLM API        |
| E2E tests | `uv run pytest tests/e2e/ -v -s`   | Serial       | Partner service     |

> **Note**: The pytest configuration is centralized in `pyproject.toml` in the project root, with `--strict-markers` enabled by default. You can flexibly switch between parallel and serial mode from the command line with `-n auto` / `-n 0`.

## Notes

1. **Integration tests** require a valid LLM configuration (each Agent's `config.toml`)
2. **E2E tests** require the Partner service to be running (each Agent has its own port; see each Agent's `config.toml`)
3. When test runs take a long time, you can use the `-x` parameter to stop at the first failure
4. Use `--tb=short` to simplify error output
5. Use `-s` to display print and logging output (recommended for E2E tests)

## State Transition Specification

**Key Principles:**

- `Rejected` = the request is **outside the service scope** (capability mismatch) and is a terminal state
- `AwaitingInput` = the request is **within scope but the information is incomplete** and requires supplementary information
- A request within scope should not enter the `Rejected` state even if its information is incomplete

## Test Fixtures

### Unit Test Fixtures (`unit/conftest.py`)

- `mock_llm_responses`: Mock LLM response factory
- `mock_openai_client_factory`: Mock OpenAI client with configurable responses
- `test_agent_dir`: Temporary test Agent directory (including configuration files)
- `message_factory`: Create test messages
- `rpc_request_factory`: Create test RPC requests

### Integration Test Fixtures (`integration/conftest.py`)

- `client`: FastAPI TestClient instance
- `online_agents`: List of all online Agent names
- `rpc_call`: Helper function that performs RPC calls
- `wait_for_state`: Wait for a task to reach a specified state
- `continue_task`: Send the CONTINUE command
- `get_task_context`: Get the task's internal context
- `assert_rpc_success`, `assert_task_state`, `assert_task_has_product`: Verification helpers

### E2E Test Fixtures (`e2e/conftest.py`)

- `http_client`: httpx HTTP client (real HTTP requests)
- `available_agents`: List of available Agents
- `unique_ids`: Generate unique task_id and session_id
- `rpc_helper`: RPC call helper class providing the start/get/continue_task/complete/cancel/poll_until methods
