**[English](README_en.md) | [中文](README.md)**

# demo-partner

demo-partner is the Partner Agent example application of ACPs; it runs in a multi-Agent manner and each Agent exposes an independent port,
used to demonstrate Partner invocation and orchestration based on the ACPs SDK. This document explains the project positioning and daily development; for full-stack packaging and deployment, see Chapter 3.

## 1. Overview

### 1.1. Project Positioning

- Provide multiple Partner Agent examples that can run independently
- Serve as the downstream service for demo-leader, discovery-server and overall chain integration
- Demonstrate the Partner operating mode based on ACS / AIC / mTLS

### 1.2. Project Features

- Multi-Agent, multi-port, configuration-driven
- No database, no dependency on PostgreSQL / Alembic
- Both local certificates and release certificates are managed per Agent in `partners/online/*/`

### 1.3. Directory Overview

```text
demo-partner/
├── partners/              # Partner business code and online configuration
├── tests/                 # unit / integration / e2e
└── Justfile               # Entry point for local development, testing and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation documentation](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation documentation](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-infra/` already exists in the sibling directory
- If the complete integration chain needs to be run, the host machine should also run `registry-server`, `ca-server`, `discovery-server`

### 2.2. Quick Start

```bash
git clone <repository-url>
cd demo-partner

# It is recommended to explicitly copy the template and check the key configuration first (if missing, just prep env / just dev start will also generate it).
cp .env.example .env
# Edit .env: confirm RabbitMQ, LLM and the runtime parameters of each Agent

just dev start
```

By default, multiple Partner Agents are started, with a port range of `9021-9025`.

### 2.3. Common Commands

```bash
# Help
just help                         # Print a command overview; running just directly also displays help

# Shared dependencies
just infra up rabbitmq            # Start the shared dependencies required by demo-partner
just infra status                 # Show shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example when missing
just prep sync                    # Download managed Python 3.14, and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep certs                   # Prepare local mTLS development certificates

# Development
just dev check                    # Read-only check of the development environment, certificates and key configuration
just dev bootstrap                # Optional: only warm up the environment, do not start services
just dev start                    # Start Partner Agents in the background
just dev start fg                 # Start in the foreground, for easier debugging
just dev logs follow              # Continuously follow logs
just dev stop                     # Stop the local instance

# Testing
just test check                   # Read-only check of the test environment
just test bootstrap               # Manually warm up the test environment; integration/e2e are implicitly run as needed
just test unit                    # Unit tests
just test integration             # Integration tests
just test e2e                     # Black-box e2e
just test                         # Runs all by default

# Packaging
just package check                # Read-only check of packaging prerequisites
just package bootstrap            # Fill in packaging prerequisites as needed (does not generate release artifacts)
just package wheel                # Build the online runtime package

# Quality
just qa                           # Show qa help
just qa precommit                 # Run the full pre-commit gate
just qa pip-audit                 # Dependency vulnerability audit
```

### 2.4. Development Notes

- The Python required to run the project does not depend on the version preinstalled on the machine; `just prep sync` downloads managed Python 3.14 via `uv`,
  and installs dependencies into the `.venv/` of the current project.
- `partners/online/*/acs.json` defines Agent capabilities, and `config.toml` defines the runtime configuration.
- `just prep certs` generates local mTLS certificates according to the AIC declaration of each Agent; these temporary files should not be committed to Git.
- `just dev start` automatically completes environment preparation before starting (you can also run `just dev bootstrap` separately to only warm up without starting).
- Integration tests and e2e depend on RabbitMQ; for complete business chain verification, `demo-leader` usually also needs to be started at the same time.
- Deployed instances can also be provided to e2e tests for reuse through `TEST_E2E_BASE_URLS`.

## 3. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installation package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the installation package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses the `just` / `scripts/` of this repository. Only when component artifacts need to be produced manually for collection by the installation layer should the `just package wheel` of this repository be used (see `justfile`, not elaborated here).
