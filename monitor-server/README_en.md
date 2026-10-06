**[English](README_en.md) | [中文](README.md)**

# monitor-server

monitor-server is the monitoring service of ACPs, responsible for collecting, storing, and querying AMP (Agent Monitoring Protocol) monitoring logs.
This document focuses on the project positioning, day-to-day development, and testing instructions.

## 1. Overview

### 1.1. Project Positioning

- Consumes the Kafka `amp.audit` topic, verifies signatures, builds a hash chain, and performs idempotent writes to PostgreSQL
- Provides a FastAPI Query API externally, supporting filtering, sorting, and cursor-based pagination queries as defined by the AMP spec

### 1.2. Project Characteristics

- Dual-link architecture: the write link (Kafka Consumer) and the query link (FastAPI) run independently
- Local development consistently uses "host processes + `../acps-infra/dev-infra`", including PostgreSQL and Kafka (Redpanda)
- Log forwarding is handled by Fluent Bit (`config/fluent-bit/`) or a Python tail script, decoupled from the application process

### 1.3. Directory Overview

```text
monitor-server/
├── app/                  # Business code (writer / query / audit / core)
├── alembic/              # Database migrations
├── config/               # Runtime configuration (TOML + Fluent Bit)
├── tests/                # unit / integration / e2e
├── scripts/              # Operations and debugging scripts
├── plans/                # Build and requirements plans (not committed)
└── Justfile              # Entry point for local development, testing, and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation documentation](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation documentation](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-infra/` already exists in the sibling directory
- Fluent Bit is already installed on this machine (`brew install fluent-bit`)

### 2.2. Quick Start

```bash
git clone <repository URL>
cd monitor-server

# It is recommended to explicitly copy the template and check the key configuration first (just prep env / just dev start will also generate it when it is missing).
cp .env.example .env
# Edit .env: confirm sensitive items such as the database and Kafka addresses

just dev start
```

Commonly used addresses after startup:

- Query API: `http://localhost:9009`
- OpenAPI documentation: `http://localhost:9009/docs`
- Health: `http://localhost:9009/health`

### 2.3. Common Commands

```bash
# Help
just help                         # Print the command overview; running just directly also shows help

# Shared dependencies
just infra up postgres kafka      # Start the shared dependencies required by monitor-server
just infra status                 # Show shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example when missing
just prep sync                    # Download managed Python 3.14 and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep migrate dev             # Migrate the development database
just prep migrate test            # Migrate the test database

# Development
just dev check                    # Read-only check of the development environment, database, Kafka, and key configuration
just dev bootstrap                # Optional: only pre-warm the environment without starting services
just dev start                    # Start services in the background
just dev start fg                 # Start in the foreground for easier debugging
just dev logs follow              # Continuously follow logs
just dev stop                     # Stop the local instance

# Testing
just test check                   # Read-only check of the test environment
just test bootstrap               # Manually pre-warm the test environment; integration/e2e/coverage run implicitly on demand
just test unit                    # Unit tests
just test integration             # Integration tests
just test e2e                     # Black-box E2E
just test coverage                # Generate coverage statistics
just test                         # Runs all by default, executing unit / integration / e2e in order

# Packaging
just package check                # Read-only check of packaging prerequisites
just package bootstrap            # Fill in packaging prerequisites on demand (does not produce release artifacts)
just package wheel                # Build the online runtime package

# Quality
just qa                           # Show qa help
just qa precommit                 # Run the full pre-commit gate
just qa pip-audit                 # Dependency vulnerability audit
```

### 2.4. Development Notes

- The Python required to run the project does not depend on the version preinstalled on this machine; `just prep sync` downloads managed Python 3.14 via `uv`,
  and installs the dependencies into the current project's `.venv/`.
- Before `just dev start` starts, it automatically prepares `.venv`, hooks, and development database migrations (including waiting for Kafka readiness); you can also run `just dev bootstrap` separately to only pre-warm without starting.
- Log forwarding: the development environment uses Fluent Bit, with the configuration file at `config/fluent-bit/fluent-bit.conf`;
  on macOS you must keep `Workers 1` in the `[OUTPUT]` section, otherwise the Kafka plugin will exit silently.
- `AUDIT_SIGNING_PRIVATE_KEY` needs to match the public key pair of demo-leader / demo-partner;
  only after alignment can audit log signature verification pass.
- To verify the complete audit pipeline (log output→Kafka→storage→query), please refer to [docs/local-audit-e2e.md](docs/local-audit-e2e.md);
  that document also covers the Mock signature verification mode (default, purely local) and the CA joint signature verification mode (connecting to ca-server, production-grade).

## 3. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installation package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repo wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble installation package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses this repository's `just` / `scripts/`. Only when you need to manually produce this component's artifacts for the installation layer to collect should you use this repository's `just package …` / `scripts/release-app/` (see `justfile` and `scripts/`; not covered here).

## 4. Related Documentation

- [AMP specification](../acps-specs/09-ACPs-spec-AMP/ACPs-spec-AMP_en.md)
- For the AMP design and the Audit API design, please refer to the internal design materials.
