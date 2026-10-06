**[English](README_en.md) | [中文](README.md)**

# Leader Agent Platform

Leader Agent Platform is an intelligent agent collaboration platform based on the **"universal foundation + scenario plugins"** design philosophy. As the core coordinator (Leader), it is responsible for receiving user requests, dynamically loading scenario configurations, orchestrating multiple Partner Agents to work together, and finally integrating the results and delivering them to the user. Communication follows AIP (Agent Interaction Protocol), supporting both Direct RPC and Group (message queue) modes.

## 1. Directory Structure

```
leader/
├── main.py                    # Application entry point (FastAPI)
├── config.toml                # Platform-level system configuration (LLM Profile, ports, discovery service, etc.)
├── acs.json                   # Leader's own ACS definition
├── assistant/                 # Platform core code
│   ├── config.py              # Configuration loading
│   ├── api/                   # HTTP interface layer (/api/v1/submit, /api/v1/result, etc.)
│   │   ├── routes.py          # Route definitions
│   │   └── schemas.py         # Request/response data models
│   ├── core/                  # Core orchestration and business logic
│   │   ├── orchestrator.py    # Main orchestrator (chaining intent analysis → task planning → execution → integration)
│   │   ├── session_manager.py # Session lifecycle management
│   │   ├── intent_analyzer.py # Intent analysis (LLM-1)
│   │   ├── planner.py         # Full planning and dimension decomposition (LLM-2)
│   │   ├── input_router.py    # User supplementary input dispatch (LLM-4)
│   │   ├── completion_gate.py # Output confirmation and conflict detection (LLM-5)
│   │   ├── aggregator.py      # Final result integration (LLM-6)
│   │   ├── clarification_merger.py  # Clarification merging (LLM-3)
│   │   ├── history_compressor.py    # Historical context compression (LLM-7)
│   │   ├── executor.py              # Direct RPC mode task execution
│   │   ├── group_executor.py        # Group mode task execution
│   │   ├── group_manager.py         # Group mode group management
│   │   └── task_execution_manager.py # Task execution management
│   ├── llm/                   # LLM call wrapper
│   │   ├── client.py          # OpenAI-compatible client
│   │   └── schemas.py         # LLM-related data models
│   ├── models/                # Data model definitions
│   │   ├── session.py         # Session model
│   │   ├── task.py            # Task model
│   │   ├── partner.py         # Partner model
│   │   ├── intent.py          # Intent analysis model
│   │   ├── aip.py             # AIP protocol-related models
│   │   └── exceptions.py      # Exception definitions
│   └── services/              # External service integration
│       ├── scenario_loader.py # Scenario configuration loading
│       └── discovery_client.py # ADP discovery service client
└── scenario/                  # Scenario configuration collection
    ├── base/                  # Base persona + general scenarios (fallback behavior)
    │   └── prompts.toml
    ├── expert/                # Professional scenario plugins
    │   ├── tour/              # Travel scenario
    │   │   ├── domain.toml    # Scenario metadata (dimension definitions, consistency rules, etc.)
    │   │   ├── prompts.toml   # Scenario prompts (Intent/Planning/Aggregation)
    │   │   └── *.json         # Static ACS files (Partner capability descriptions)
    │   └── divination/        # Other scenario examples
    │       ├── domain.toml
    │       └── prompts.toml
    └── offline/               # Editing area (used for atomic updates of configuration)
```

### Core Design

- **Universal foundation**: The code under the `assistant/` directory provides general capabilities such as Session management, AIP protocol communication, task state machine and LLM call wrappers.
- **Scenario plugins**: The `scenario/` directory defines domain-specific intent analysis, task decomposition, Partner selection and result integration logic through configuration files. Adding a new scenario only requires adding a configuration directory under `scenario/expert/`, with no need to modify platform code.

## 2. Configuration

### config.toml

Platform-level system configuration, containing the following main sections:

| Configuration Section  | Description                                                |
| -------------------- | ---------------------------------------------------------- |
| `[app]`              | Leader ACS description file path; the program parses `aic` from it as its own identity |
| `[logging]`          | Global log level                                           |
| `[logging.packages]` | Log level override entries for specified packages and their submodules |
| `[uvicorn]`          | HTTP service listen address, port and hot reload switch    |
| `[mtls]`             | Client certificate configuration used when the Leader accesses Partner HTTPS endpoints |
| `[llm.*]`            | LLM Profile configuration; the example contains three groups: `fast`, `default`, `pro` |
| `[discovery]`        | ADP discovery service address, timeout and returned quantity limit |
| `[group]`            | Group mode switch, status probing, wait timeout and retry parameters |
| `[rabbitmq]`         | RabbitMQ connection parameters, used for Group mode message communication |

`config.toml` has been committed to the repository as formal configuration. Actual LLM parameters are injected through the `.env` in the repository root; the default OIDC configuration for local development is stored directly in
`config.toml` and `web_app/runtime-config.js`, so that it is aligned out of the box with the Keycloak of `acps-infra/dev-infra`. It is recommended to use the repository root's
`just prep env` or `just dev start` to automatically prepare the local environment:

```bash
just prep env
```

At least the following variables need to be filled in:

- `LEADER_LLM_FAST_API_KEY`
- `LEADER_LLM_FAST_BASE_URL`
- `LEADER_LLM_FAST_MODEL`
- `LEADER_LLM_DEFAULT_API_KEY`
- `LEADER_LLM_DEFAULT_BASE_URL`
- `LEADER_LLM_DEFAULT_MODEL`
- `LEADER_LLM_PRO_API_KEY`
- `LEADER_LLM_PRO_BASE_URL`
- `LEADER_LLM_PRO_MODEL`

The `[mtls]` section fixedly references `atr/client.pem`, `atr/client.key` and `atr/trust-bundle.pem`; the certificate script only overwrites the file contents and no longer rewrites `config.toml`.

In `leader/config.toml`, environment variable name references in the `llm.*.*_env` form are kept only for LLM sensitive information, to avoid writing specific keys into the repository; ports,
and stable local development defaults such as OIDC issuer / audience / algorithm are committed directly to the repository for maintenance.

### Scenario Configuration

- `scenario/base/prompts.toml`: defines the base persona and the prompts under general scenarios.
- `scenario/expert/<scenario name>/domain.toml`: defines scenario metadata, including dimension decomposition logic, Partner mapping relationships and cross-dimension consistency rules.
- `scenario/expert/<scenario name>/prompts.toml`: defines scenario-specific Intent / Planning / Aggregation prompts.
- `scenario/expert/<scenario name>/*.json`: static ACS files, describing the Partner capabilities preferentially used by that scenario.

## 3. Running

Run from the repository root (recommended):

```bash
# Start Leader API + Web UI in the background (the environment is prepared automatically before starting)
just dev start

# Only observe Leader API startup logs in foreground mode
just dev start fg
```

Or start directly through uvicorn:

```bash
uv run uvicorn leader.main:app --host 0.0.0.0 --port 9031 --reload
```

> **Note**: The Leader depends on local integration services such as Discovery, MQ Auth and Partner; the default addresses are in the repository root `.env.example`. Partner can be started through the `just dev` workflow of the sibling project `demo-partner`.

## 4. API Interfaces

The Leader provides API interfaces for the Web frontend to call; the following are the main HTTP interfaces:

| Method | Path                          | Description                     |
| ---- | ----------------------------- | ----------------------------- |
| POST | `/api/v1/submit`              | Submit a user request (create/continue a session) |
| GET  | `/api/v1/result/{session_id}` | Poll task status and results  |

### Interaction Flow

1. The client calls `POST /api/v1/submit` to submit user input, and obtains `sessionId` and `activeTaskId`.
2. The client uses `GET /api/v1/result/{session_id}` to poll for results.
3. Response statuses include: `pending`, `running`, `awaiting_input` (clarification), `completed`, `failed`.
4. Upon receiving `awaiting_input`, the client calls `/submit` again (with `sessionId`) to submit supplementary information.
