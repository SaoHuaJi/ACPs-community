**[English](README_en.md) | [中文](README.md)**

# ca-server

ca-server is the certificate service of ACPs, responsible for certificate application, issuance, revocation and query in ATR scenarios. This document explains the project positioning and daily development; for full-stack packaging and deployment, see Chapter 3.

## 1. Overview

### 1.1. Project Positioning

- Provide ACME, CRL, OCSP and trust bundle related protocol endpoints externally
- Issue business certificates for Agents, and maintain certificate status and revocation information
- Provide internal/admin certificate management interfaces for other ACPs components

### 1.2. Project Features

- Simultaneously carry three types of protocol capabilities: ACME, CRL and OCSP
- Local development certificate materials are uniformly issued by `../acps-infra/dev-infra`
- The default development mode can mock `registry-server`, and switch to real integration when needed

### 1.3. Directory Overview

```text
ca-server/
├── app/                         # Business code
├── alembic/                     # Database migrations
├── certs/                       # Local development certificate materials
├── tests/                       # unit / integration / e2e
└── Justfile                     # Entry point for local development, testing and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation documentation](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation documentation](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-infra/` already exists in the sibling directory

### 2.2. Quick Start

```bash
git clone <repository-url>
cd ca-server

# It is recommended to explicitly copy the template and check the key configuration first (if missing, just prep env / just dev start will also generate it).
cp .env.example .env
# Edit .env: confirm sensitive items such as the database, CA material paths, service tokens

just dev start
```

Common addresses after startup:

- API: `http://localhost:9003`
- Docs: `http://localhost:9003/docs`
- Health: `http://localhost:9003/health`

### 2.3. Common Commands

```bash
# Help
just help                         # Print a command overview; running just directly also displays help

# Shared dependencies
just infra up postgres            # Start the shared dependencies required by ca-server
just infra status                 # Show shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example when missing
just prep sync                    # Download managed Python 3.14, and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep certs                   # Prepare local CA development materials
just prep migrate dev             # Migrate the development database
just prep migrate test            # Migrate the test database

# Development
just dev check                    # Read-only check of the development environment, certificates and key configuration
just dev bootstrap                # Optional: only warm up the environment, do not start services
just dev start                    # Start services in the background
just dev start fg                 # Start in the foreground, for easier debugging
just dev logs follow              # Continuously follow logs
just dev stop                     # Stop the local instance

# Testing
just test check                   # Read-only check of the test environment
just test bootstrap               # Manually warm up the test environment; integration/e2e/coverage are implicitly run as needed
just test unit                    # Unit tests
just test integration             # Integration tests
just test e2e                     # Black-box e2e
just test coverage                # Generate coverage statistics
just test                         # Runs all by default, executing unit / integration / e2e in sequence

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
- `just prep certs` exports the CA suite required by `ca-server` from the shared development PKI.
- `just dev start` automatically completes environment preparation before starting (you can also run `just dev bootstrap` separately to only warm up without starting).
- In development mode, the real `registry-server` is not requested by default; if real integration is needed, change
  `[registry_server].mock` to `false` in `config/development.toml`, or temporarily inject `REGISTRY_SERVER_MOCK=false` as an override before the startup command.
- The `tests/e2e/` of this repository only verifies the black-box behavior of `ca-server` itself; for cross-service integration, go to `acps-cli/tests/e2e/`.
- The certificate external addresses, OCSP and CRL addresses in the production configuration should be confirmed before deployment; relying on default placeholder values is not recommended.

## 3. Packaging and Deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installation package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the installation package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses the `just` / `scripts/` of this repository. Only when component artifacts need to be produced manually for collection by the installation layer should the `just package wheel` of this repository be used (see `justfile`, not elaborated here).
