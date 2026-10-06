**[English](README_en.md) | [中文](README.md)**

# demo-leader

demo-leader is the ACPs Leader Agent sample application; it is responsible for receiving user input, orchestrating Partner Agents, aggregating results,
and outputting them as API / SSE and a Web UI. This document explains the project positioning and day-to-day development; full-stack packaging and deployment are covered in Chapter 3.

## 1. Overview

### 1.1. Project Positioning

- Provides the Leader API, responsible for receiving requests, orchestrating Partners and aggregating results
- Provides a local Web UI for manual integration testing and demonstrations
- Serves as the upstream business entry point for demo-partner, mq-auth-server, registry / ca / discovery

### 1.2. Project Features

- Dual-process operation: `9031` Leader API, `9030` Web UI
- No database; business state is mainly maintained by orchestration logic and session flow
- `leader/config.toml` keeps non-sensitive configuration; sensitive values uniformly go through environment variables
- Runtime mTLS certificates are located in `leader/atr/`

### 1.3. Directory Overview

```text
demo-leader/
├── leader/                      # Leader API, runtime configuration, ACS and scenarios
├── web_app/                     # Local static frontend
├── tests/                       # unit / integration / e2e
├── scripts/smoke-test-business.sh
└── Justfile                     # Entry point for local development, testing and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation documentation](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation documentation](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-sdk/`, `../acps-cli/`, `../acps-infra/` already exist in the sibling directory
- If you need to run integration tests or e2e, you usually also need to start the sibling `demo-partner` first

### 2.2. Quick Start

```bash
git clone <repository URL>
cd demo-leader

# It is recommended to explicitly copy the template and check the key configuration first (just prep env / just dev start will also generate it when missing).
cp .env.example .env
# Edit .env: fill in LLM sensitive information; non-sensitive defaults such as local OIDC / Web ports have been consolidated into the repository configuration

just dev start
```

Common addresses after startup:

- Web UI: `http://localhost:9030`
- Leader API: `http://localhost:9031`

### 2.3. Common Commands

```bash
# Help
just help                         # Output the command overview; running just directly also shows help

# Shared dependencies
just infra status                 # Check shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example when missing
just prep sync                    # Download managed Python 3.14 and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep certs                   # Generate local certificates based on leader/atr/acs.json

# Development
just dev check                    # Read-only check of the development environment, certificates, sibling prerequisites and key configuration
just dev bootstrap                # Optional: only warm up the environment without starting services
just dev start                    # Start Leader API + Web UI in the background
just dev status                   # Check background process status
just dev logs follow              # Continuously follow logs
just dev stop                     # Stop the local instance

# Testing
just test check                   # Read-only check of the test environment
just test bootstrap               # Manually warm up the test environment; api/integration/e2e will implicitly run as needed
just test unit                    # Unit tests
just test api                     # API-level tests
just test integration             # Integration tests
just test e2e                     # Black-box e2e
just test coverage                # Unit test coverage
just test                         # Runs all by default, executing unit / api / integration / e2e in sequence

# Packaging
just package check                # Read-only check of packaging prerequisites
just package bootstrap            # Fill in packaging prerequisites on demand (does not generate release artifacts)
just package wheel                # Build the online wheel runtime package

# Quality
just qa                           # Show qa help
just qa precommit                 # Run the full pre-commit gate
just qa pip-audit                 # Dependency vulnerability audit
```

### 2.4. Development Notes

- The Python required to run the project does not depend on the version pre-installed on the machine; `just prep sync` downloads managed Python 3.14 through `uv`,
  and installs the dependencies into the current project's `.venv/`.
- The local mTLS certificates under `leader/atr/` are generated by `just prep certs` and should not be committed to Git.
- `just dev start` automatically completes environment preparation before starting (you can also run `just dev bootstrap` separately to only warm up without starting).
- Integration tests and e2e usually require the sibling `demo-partner` to be started first.
- Sensitive information such as real LLM keys is injected via `.env`; the default OIDC configuration used for local development has been committed in
  `leader/config.toml` and `web_app/runtime-config.js`, and is aligned with the Keycloak of `acps-infra/dev-infra`.
- The Web UI serves local debugging and demonstrations by default; on the standalone delivery side the port is still controlled by `acps-infra`'s `LEADER_WEB_PORT`.

## 3. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installation package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assembling the installation package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration testing still uses this repository's `just` / `scripts/`. Only when you need to manually produce this component's artifacts for the installation layer to collect should you use this repository's `just package wheel` (see `justfile`, not expanded here).
