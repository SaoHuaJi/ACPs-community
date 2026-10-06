**[English](README_en.md) | [中文](README.md)**

# acps-cli

`acps-cli` is the unified command-line toolset of ACPs, providing five categories of client capabilities: Registry, CA, Discovery, MQ, and Monitor, oriented toward development integration, debugging and verification, install-layer provision bootstrap, and daily operations scripts.

## 1. Overview

### 1.1. Project Positioning

This project is a pure CLI tool; it does not start FastAPI, database, or message queue services. Whether for local development or generic packaging deployment, integration must be completed through external backend services.

Main capabilities include:

- Registry user side: login, logout, change password, register or update Agent, submit for review, obtain EAB, sync ACS
- Registry admin side: login, logout, change password, review, enable/disable Agent, reset the password of a specified user
- CA client: apply for, renew, and revoke certificates, rotate ACME account keys, check certificate status
- Discovery client: trigger DSP sync, execute queries, check service health status
- MQ client: check mq-auth-server health status, manage Group ACL, and probe Auth API allow / deny decisions
- Monitor client: query the heartbeat, metrics, access, message, system, and audit read models of monitor-server

### 1.2. Commands and Documentation

The current unified entry point is `acps-cli`, and the main command domains are as follows:

- `acps-cli auth` / `agent` / `entity`: Registry user-side operations
- `acps-cli cert`: certificate lifecycle and EAB related operations
- `acps-cli discover`: Discovery queries and status viewing
- `acps-cli admin registry ...`: Registry admin-plane commands
- `acps-cli admin ca ...`: CA admin-plane commands
- `acps-cli admin discovery ...`: Discovery admin-plane commands
- `acps-cli admin mq ...`: mq-auth-server admin-plane commands
- `acps-cli monitor ...`: monitor-server query commands

All CLIs support:

- `--config PATH`: explicitly specify `acps-cli.toml`
- `--verbose`: output DEBUG logs; by default output INFO and above logs

Where an address needs to be temporarily overridden per service domain, `--server-url` can be used on the corresponding command group; Registry-related commands additionally support `--timeout`.

## 2. Development

### 2.1. Development Environment and Prerequisites

This project is usually set up together with the following sibling repositories:

```text
acps/
  acps-infra/
  registry-server/
  ca-server/
  discovery-server/
  mq-auth-server/
  monitor-server/
  acps-cli/
```

- `uv` ([installation documentation](https://docs.astral.sh/uv/getting-started/installation/)) — `uv` will automatically download and manage Python 3.14 according to `.python-version`, so there is no need to install Python manually
- `just` ([official installation documentation](https://just.systems/man/en/packages.html))
- Docker Desktop (only used to start the `acps-infra/dev-infra` dependencies)
- `../acps-infra/` and the five backend sibling repositories already exist in the same-level directory

Additional note: development in this repository uniformly uses Python `3.14`, pinned through the `.python-version` version request at the repository root; `just dev bootstrap` will force the use of managed Python `3.14` through `uv` to create and sync `.venv`.

### 2.2. Setting Up the CLI Development Environment

`acps-cli` is a pure CLI tool with no locally long-running service processes; the public main development path uses `just dev check` and `just dev bootstrap`, which are responsible for preparing the CLI's own runtime environment and the shared `dev-infra` dependencies. Run it once when first entering the repository; `just test integration` / `just test e2e` will also fill in test prerequisites as needed.

```bash
just dev bootstrap   # Set up the CLI development environment and shared dependencies
```

`just dev bootstrap` performs operations such as `infra up postgres/redis/rabbitmq + prep env + prep sync + prep hooks`.

Development configuration conventions:

- The repository root already provides `acps-cli.toml` as the default local development configuration; please adjust the service addresses in it according to your actual situation, ensuring they can connect to backend service instances.
- `registry.auth.mode = "local" | "oidc"`: `local` still uses username/password; `oidc` is fixed to Device Authorization Grant and no longer accepts `--username/--password`
- `monitor.auth.mode = "none" | "oidc"`: `none` only performs anonymous queries in development mode; `oidc` requires running `monitor auth login` first
- If OIDC is used, provide `issuer` / `client_id` in `acps-cli.toml` or in environment variables, and log in separately by service domain: `auth login` / `admin auth login` / `monitor auth login`
- If a default display name and organization name need to be provided for auto-register, set `display_name` / `org_name` in `[registry]` of `acps-cli.toml`
- `just prep env` can still be used to generate a `.env` placeholder file for future environment variable override scenarios
- The specific command tree can be viewed through `acps-cli --help`, `acps-cli cert --help`, `acps-cli admin --help`

If you are currently only modifying the CLI itself, viewing help, or debugging a single command, there is no need to start the five sibling services first; only when real integration is needed should you proceed to the next section.

### 2.3. Starting the Local Integration Environment

When you need to manually integrate the five backend services Registry, CA, Discovery, MQ, and Monitor, first execute in the `acps-cli` repository:

```bash
just dev bootstrap
```

Then start the five backend services separately in **independent terminals**. Please configure these application services yourself to ensure they can connect to one another; the following are command examples:

```bash
# Terminal 1
cd ../registry-server && APP_ENV=development CA_SERVER_MOCK=false just dev start

# Terminal 2
cd ../ca-server && APP_ENV=development REGISTRY_SERVER_MOCK=false just dev start

# Terminal 3
cd ../discovery-server && just dev start

# Terminal 4
cd ../mq-auth-server && just dev start

# Terminal 5
cd ../monitor-server && APP_ENV=development DATABASE_URL=postgresql+asyncpg://monitor:monitor@localhost:5432/agent_monitor_test TEST_DATABASE_URL=postgresql+asyncpg://monitor:monitor@localhost:5432/agent_monitor_test REDIS_URL=redis://localhost:6379/3 CLICKHOUSE_DATABASE=amp_test OPENSEARCH_HOSTS=http://localhost:9200 OPENSEARCH_VERIFY_CERTS=false just dev start
```

During integration, it is recommended to pay attention to the following points:

- When integrating `registry-server`, it is recommended to use `APP_ENV=development CA_SERVER_MOCK=false` to enable the real CA revocation notification chain.
- `registry-server` and `ca-server` need to use the same `REGISTRY_SERVER_INTERNAL_API_TOKEN`; the examples above uniformly use `local-registry-server-internal-api-token`.
- The `.env` of `ca-server` must set a non-empty `CA_SERVER_ADMIN_API_TOKEN` (the local default value in `.env.example` is `local-ca-admin-token`), otherwise admin endpoints such as CRL refresh will return authentication not configured; CLI integration/tests also use this local default value by default.
- Both `9007` / `9008` of `mq-auth-server` require mTLS; during local integration please ensure `../mq-auth-server/certs/` or your own `[mq]` client certificate configuration is already in place — `just dev check` will check MQ together with the other three services.
- If `monitor-server` needs to share the same local environment with monitor live integration / e2e, it is recommended to directly reuse `agent_monitor_test` / Redis DB 3 / ClickHouse `amp_test` / local OpenSearch, and to explicitly override these environment variables before `just dev start`; at startup, `config/audit_keys.json` (mock audit signature verification keys) will be generated when missing.

After the five services have finished starting, return to the `acps-cli` repository and execute:

```bash
just dev check
```

If `check` fails, it will clearly tell you which HTTP service is missing and print the corresponding repository's start command.

Commonly used local addresses:

| Service                  | Address                  | Description                          |
| ------------------------ | ------------------------ | ------------------------------------ |
| registry-server          | `http://localhost:9001`  | `acps-cli.toml` `[registry]` direct connection |
| ca-server                | `http://localhost:9003`  | `acps-cli.toml` `[ca]` direct connection |
| discovery-server         | `http://localhost:9005`  | `acps-cli.toml` `[discovery]` direct connection |
| mq-auth-server Group API | `https://localhost:9007` | `acps-cli.toml` `[mq].group_api_url` |
| mq-auth-server Auth API  | `https://localhost:9008` | `acps-cli.toml` `[mq].auth_api_url`  |
| monitor-server           | `http://localhost:9009`  | `acps-cli.toml` `[monitor].base_url` |

Integration command examples:

```bash
uv run acps-cli auth login --username alice --password 'S3cret!'
uv run acps-cli agent save --acs-file acs.json
uv run acps-cli cert status --aic <AIC>
uv run acps-cli discover query "Beijing travel recommendations"
uv run acps-cli monitor heartbeat liveness <AIC>
```

If `registry-server` / `monitor-server` have OIDC enabled, the corresponding login examples become:

```bash
uv run acps-cli auth login
uv run acps-cli admin auth login
uv run acps-cli monitor auth login
uv run acps-cli auth status --json
uv run acps-cli monitor auth status --json
```

Additional notes:

- `discover query` outputs JSON directly by default, so there is no need to append `--json`.
- If you want a stable discovery visibility gate, it is recommended to use `discover query --type filtered --filter-json ...` instead.

For example:

```bash
uv run acps-cli discover query \
  --type filtered \
  --filter-json '{"conditions":[{"field":"aic","op":"eq","value":"<partner-aic>"},{"field":"active","op":"eq","value":true}]}'
```

### 2.4. Daily Development Commands

Commonly used development commands:

```bash
# Help
just help                 # Output an overview of commands

# Development (the CLI has no long-running services)
just dev bootstrap        # Set up the CLI development environment and shared dependencies
just dev check            # Read-only check of the integration environment

# Packaging
just package check        # Read-only check of packaging prerequisites
just package bootstrap    # Fill in packaging prerequisites as needed (does not generate release artifacts)
just package wheel        # Build the online runtime package

# Quality
just qa                   # Show qa help
just qa precommit         # Execute the full pre-commit gate
just qa pip-audit         # Dependency vulnerability audit
```

## 3. Testing

`acps-cli` is the only repository, apart from the multiple server repositories, that carries real cross-service integration e2e.

### 3.1. Test Responsibilities and Layering

The test boundaries are agreed as follows:

- `registry-server`, `ca-server`, `discovery-server`, `mq-auth-server`, and `monitor-server` are each responsible for their own service's `unit`, `integration`, and self-contained `e2e`.
- Whenever a test needs to verify the real interactions of multiple sibling services at the same time, it should belong to `acps-cli/tests/e2e/` rather than remaining in a server repository.
- Typical integration scenarios include: the ATR / EAB / certificate application main chain, certificate lifecycle status propagation, discovery snapshot / incremental / webhook / runtime collaboration, and mq group / auth-probe workflows.
- A small number of scenarios explicitly marked as future work are allowed to remain `skip`; apart from that, the goal of integration tests is to be as green as possible by automatically preparing prerequisites.

| Layer              | Command                 | Description                                                                             |
| ------------------ | ----------------------- | --------------------------------------------------------------------------------------- |
| Unit tests         | `just test unit`        | Pure mock, no external service dependencies                                             |
| Integration tests  | `just test integration` | Mainly validates the CLI's own arguments, configuration, output, and single-service command contracts; when services are missing at the default local addresses, fixtures host them automatically |
| End-to-end tests   | `just test e2e`         | The main entry point for real cross-service integration; when services are missing at the default local addresses, fixtures host them automatically |
| Full test suite    | `just test`             | Run all tests                                                                           |

Responsibility division recommendations:

- `just test integration`: focuses on the CLI command surface, configuration parsing, output format, and single-service command contracts.
- `just test e2e`: focuses on cross-service user journeys, real state propagation, and integration topology collaboration.
- `just test`: runs the CLI's unit / integration / e2e in sequence; when services are missing at the default local addresses, test fixtures fill them in.

Correspondence with each server repository:

- The respective `integration` and `e2e` of `registry-server`, `ca-server`, `discovery-server`, `mq-auth-server`, and `monitor-server` are responsible for their own service's closed-loop verification.
- `acps-cli/tests/e2e/` is responsible for connecting multiple services together for real integration regression.
- If a scenario must start multiple sibling services at the same time, it should preferentially enter `acps-cli/tests/e2e/` rather than flowing back to a server repository.

### 3.2. Setting Up the Test Environment

Test entry points uniformly use:

```bash
just test check      # Read-only check of the test environment
just test bootstrap   # Manually warm up the test environment; integration / e2e / all will implicitly run it as needed
```

`just test bootstrap` and `just dev bootstrap` reuse the same shared preparation logic, but semantically it is specifically oriented toward test environment preparation. `just test unit` does not require bootstrap; integration / e2e / all will automatically fill in prerequisites when needed.

There are two ways to use the test environment:

- Automatic hosting mode: tests use the default local addresses, namely `REGISTRY_URL=http://localhost:9001`, `CA_URL=http://localhost:9003`, `DISCO_URL=http://localhost:9005`, `MQ_GROUP_API_URL=https://localhost:9007`, `MQ_AUTH_API_URL=https://localhost:9008`, `MONITOR_BASE_URL=http://localhost:9009`; only when these addresses are unreachable at test startup will `tests/_local_services.py` start the required sibling services under test-mode management.
- Manual hosting mode: at test startup, if the target addresses you configured are already reachable, regardless of whether they are the default ports, the tests will directly reuse these already-running services and will not launch new sibling processes.
- Custom address mode: if you change the above environment variables to non-default addresses or ports, the tests will treat it as a "target environment hosted by yourself"; in this case, even if the target services are unreachable, the fixtures will not start them on your behalf, but will directly report an error, requiring you to start these custom target services first.

The distinction rule can be condensed into one sentence: first look at "whether the base_url the test wants to access is a default local address", then look at "whether this address is already reachable when the test starts". Only the single combination of "default local address + currently unreachable" will enter automatic hosting.

Additional note: the current automatic hosting is not a "random port mode". What `tests/_local_services.py` starts under management is still the fixed default ports `9001/9003/9005/9007/9008/9009`; it is just that the pytest process performs the bootstrap/start of each sibling repository on your behalf.

If you choose to host services manually, it is recommended to first execute:

```bash
just dev check
```

`just dev check` will check the five services registry / ca / discovery / mq / monitor together; for MQ, it will preferentially use the `[mq]` configuration, `bootstrap-artifacts/`, or the probe certificate material in the local `../mq-auth-server/certs/`.

To make `integration` / `e2e` / `just test` more stable:

- When integrating `registry-server`, it is recommended to use `APP_ENV=development CA_SERVER_MOCK=false` to enable the real CA revocation notification chain.
- `registry-server` and `ca-server` need to use the same `REGISTRY_SERVER_INTERNAL_API_TOKEN`.
- `9007` / `9008` of `mq-auth-server` require mTLS; if the tests involve `admin mq ...`, a usable client certificate needs to be prepared in advance.
- The live integration / e2e related to `monitor-server` depends on Kafka, VictoriaMetrics, ClickHouse, and OpenSearch; under the default local addresses, these dependencies will be prepared as needed when pytest starts monitor-server under management. If you choose to host monitor-server manually, please use `just dev start` as in section 2.3 and override the corresponding test environment variables.
- If `DOCKER_CONFIG` is not explicitly set, `just test bootstrap` and pytest automatic hosting will use `acps-cli/.tmp/docker-public-config/` to pull public dev-infra images, avoiding the local Docker credential helper getting stuck; if you want to continue reusing your own Docker login state, please export `DOCKER_CONFIG` yourself first.

### 3.3. Running Tests and Conventions

Recommended execution order:

1. Execute `just dev bootstrap` in the `acps-cli` repository.
2. If tests need to be run, then execute `just test bootstrap`.
3. If you want to manually integrate or use custom service addresses, start local integration instances as in section 2.3.
4. Return to `acps-cli` and execute `just dev check` to confirm that all five targets registry / ca / discovery / mq / monitor are reachable.
5. If you use the default local addresses and these addresses currently have no service listening, you can directly run `just test integration`, `just test e2e`, or `just test`; the test fixtures will automatically host the required sibling services.
6. If you have already manually started services on the default ports, or changed to custom addresses through environment variables, the tests will only reuse these existing targets; especially when custom addresses are unreachable, the tests will not automatically pull up services.

The role of `just dev check` is to perform a centralized check of the prerequisites for manual integration; if it fails, prioritize fixing the startup matrix, port, token, or certificate issues. For test paths using the default local addresses, `tests/_local_services.py` will start sibling services under test-mode management when these default addresses are unreachable, so `just test integration` / `just test e2e` are no longer required to explicitly pass `check` first. If you change to non-default addresses, both `just dev check` and the tests will only check/use the targets you specify and will not automatically fall back to the machine's default ports.

Current stable baseline:

- Under the integration startup method agreed in the README, `just test` should be able to run unit / integration / e2e through.

Scenarios currently allowed to remain `skip` should be limited to uncovered capabilities explicitly marked in the README / tests (for example, multi-instance discovery forwarder/fallback integration and fanout aggregation); `skip`s triggered because the environment is not prepared should be resolved by fixing the environment first and then running.

## 4. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` install layer**: application release package → (image) image package → install package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry points: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Install layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the install package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses this repository's `just` / `scripts/`. Only when this component's artifacts need to be produced manually for the install layer to collect should this repository's `just package wheel` be used (see `justfile`, not elaborated here).
