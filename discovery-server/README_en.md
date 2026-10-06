**[English](README_en.md) | [中文](README.md)**

# discovery-server

discovery-server is the discovery service of ACPs. It receives natural language requests, maintains the local
index, and returns available Agents based on DSP synchronization results. This document explains the project
positioning and daily development; full-stack packaging and deployment are covered in Chapter 3.

## 1. Overview

### 1.1. Project Positioning

- Receive natural language discovery requests and return candidate Agents
- Keep ACS data in sync with `registry-server` through DSP
- Maintain the embedding index, availability status, and optional forwarder logic locally

### 1.2. Project Features

- A single service supports both CPU / GPU runtime tiers
- Supports sample data import, making it easy to verify discovery behavior within a single repository
- Detailed runtime behavior and advanced switches are kept uniformly in `config/default.toml` and `config/{APP_ENV}.toml`

### 1.3. Directory Overview

```text
discovery-server/
├── app/                  # discovery / sync / core
├── config/               # TOML layered configuration
├── alembic/              # database migrations
├── tests/                # unit / integration / e2e
└── Justfile              # entry point for local development, testing, and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation documentation](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation documentation](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-infra/` already exists as a sibling directory
- `../acps-sdk/` already exists as a sibling directory
- If sample data needs to be imported, it is recommended that `../demo-partner/` exists as a sibling directory, with `../demo-leader/` optional

### 2.2. Quick Start

```bash
git clone <repository URL>
cd discovery-server

# It is recommended to explicitly copy the template and check the key configuration first (just prep env / just dev start will also generate it when missing).
cp .env.example .env
# Edit .env: confirm sensitive items such as database, LLM, embedding

just dev start
```

Common addresses after startup:

- API: `http://localhost:9005`
- Docs: `http://localhost:9005/docs`
- Health: `http://localhost:9005/health`

### 2.3. Common Commands

```bash
# Help
just help                             # Print the command overview; running just directly also shows help

# Shared dependencies
just infra up postgres                # Start the shared dependencies required by discovery-server
just infra status                     # Check shared dependency status

# Environment preparation
just prep env                         # Generate .env from .env.example when missing
just prep sync                        # Download managed Python 3.14 and sync dependencies into .venv/
just prep hooks                       # Install/update Git hooks
just prep migrate dev                 # Migrate the development database
just prep migrate test                # Migrate the test database
just prep seed test                   # Import demo ACS samples into the test database (if needed)

# Development
just dev check                        # Read-only check of the development environment and key configuration
just dev bootstrap                    # Optional: only warm up the environment, do not start the service
just dev start                        # Start the service in the background
just dev start fg                     # Start in the foreground for easier debugging
just dev logs follow                  # Continuously follow the logs
just dev stop                         # Stop the local instance

# Testing
just test check                       # Read-only check of the test environment
just test bootstrap                   # Manually warm up the test environment; integration/e2e/coverage implicitly run it as needed
just test unit                        # Unit tests
just test integration                 # Integration tests
just test e2e                         # Black-box e2e
just test coverage                    # Generate coverage statistics
just test                             # Runs all by default, executing unit / integration / e2e in order

# Packaging
just package check                    # Read-only check of packaging prerequisites
just package bootstrap                # Fill in packaging prerequisites on demand (does not produce release artifacts)
just package wheel                    # Build the online runtime package

# Quality
just qa                               # Show qa help
just qa precommit                     # Run the full pre-commit gate
just qa pip-audit                     # Dependency vulnerability audit
```

### 2.4. Development Notes

- The Python required to run the project does not depend on a version preinstalled on the machine; `just prep sync`
  downloads managed Python 3.14 through `uv` and installs the dependencies into the current project's `.venv/`.
- `DISCOVERY_MODE` controls the runtime tier; CPU uses a remote embedding API, GPU uses a local model path. The
  default development environment is CPU-only: `uv sync` (or `just prep sync`) does not install the `gpu` extra by
  default, and together with the default `DISCOVERY_MODE=cpu` it can start. When you need to run GPU mode locally
  (local BGE-M3 embedding/reranker), explicitly run `uv sync --extra gpu`, and set `DISCOVERY_MODE=gpu`,
  `EMBEDDING_MODEL_PATH`, `EMBEDDING_DEVICES`, `RERANKER_URL`, and other configuration; when the `gpu` extra is
  missing, starting GPU mode gives a clear `RuntimeError` telling you to install `uv sync --extra gpu`, instead of a
  bare `ModuleNotFoundError`.
- `just test bootstrap` (as well as `just test check`) forcibly ensures that the test environment has the `gpu` extra
  installed: the standard `integration`/`e2e`/`all` entry points run the CPU and GPU modes in order by default, and
  the test environment does not distinguish between CPU/GPU developer identities.
- `just dev start` automatically completes environment preparation before starting (you can also run `just dev
  bootstrap` on its own to only warm up without starting).
- `DISCOVERY_BUILD_PROFILE` affects the dependency tier used when building the application release package / image;
  the actual runtime mode is still determined by `[discovery].mode` in `config/{APP_ENV}.toml` or the environment
  variable `DISCOVERY_MODE`.
- `just prep seed app` / `just prep seed test` read the sibling `demo-partner`, and as needed the ACS JSON of
  `demo-leader`, to generate sample data.
- For runtime behavior with many configuration items, such as forwarder, polling, secondary instance, and detailed
  API boundaries, `config/default.toml` and the corresponding environment TOML are the single source of truth; the
  root README no longer expands on them.
- Real cross-service integration testing should still move to `acps-cli/tests/e2e/`; the tests in this repository
  mainly cover the boundaries of `discovery-server` itself.

## 3. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installation package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the installation package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses this repository's `just` / `scripts/`. Only when you need to manually produce this component's artifacts for the installation layer to collect should you use this repository's `just package wheel` (see `justfile`, not expanded here).
