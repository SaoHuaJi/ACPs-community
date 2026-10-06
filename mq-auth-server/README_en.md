**[English](README_en.md) | [中文](README.md)**

# mq-auth-server

mq-auth-server is ACPs' RabbitMQ authentication and group ACL service. It provides RabbitMQ with an HTTP auth backend
and also provides the group management API for Leaders. This document explains the project positioning and day-to-day development; full-stack packaging and deployment are covered in Chapter 3.

## 1. Overview

### 1.1. Project positioning

- `9007` provides the Group API, for Leaders to manage group ACLs over mTLS
- `9008` provides the Auth API, for RabbitMQ to make allow / deny authorization decisions
- Uses Redis to store group ACLs, and calls the RabbitMQ Management API on demand to disconnect clients

### 1.2. Project characteristics

- Dual-listener architecture: the Group API and the Auth API run on separate ports
- No database; ACLs are stored entirely in Redis
- Both ports require real mTLS certificates

### 1.3. Directory overview

```text
mq-auth-server/
├── app/                  # API, services and infrastructure
├── config/               # Layered TOML configuration
├── certs/                # Local development certificates
├── tests/                # unit / integration / e2e
└── Justfile              # Entry point for local development, testing and quality checks
```

## 2. Development

### 2.1. Prerequisites

- [uv official installation docs](https://docs.astral.sh/uv/getting-started/installation/)
- [just official installation docs](https://just.systems/man/en/packages.html)
- [Docker Desktop official download](https://www.docker.com/products/docker-desktop/)
- `../acps-infra/` already exists in the sibling directory

### 2.2. Quick start

```bash
git clone <repository URL>
cd mq-auth-server

# It is recommended to copy the template explicitly first and check the key settings (just prep env / just dev start will also generate it if missing).
cp .env.example .env
# Edit .env: confirm sensitive items such as Redis, RabbitMQ and certificate paths

just dev start
```

Common addresses after startup:

- Group API: `https://localhost:9007`
- Auth API: `https://localhost:9008`

### 2.3. Common commands

```bash
# Help
just help                         # Print a command overview; running just alone also shows help

# Shared dependencies
just infra up redis rabbitmq      # Start the shared dependencies required by mq-auth-server
just infra status                 # Show shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example if missing
just prep sync                    # Download managed Python 3.14 and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep certs                   # Prepare development certificates
just prep certs reset             # Clean local certificates and re-issue them
just prep migrate test            # No-database project, explicitly skipped

# Development
just dev check                    # Read-only check of the development environment, Redis, RabbitMQ, certificates and key settings
just dev bootstrap                # Optional: only warm up the environment, without starting services
just dev start                    # Start both listeners in the background
just dev start fg                 # Start in the foreground, convenient for debugging
just dev logs follow              # Continuously follow the logs
just dev stop                     # Stop the local instance

# Testing
just test check                   # Read-only check of the test environment
just test bootstrap               # Manually warm up the test environment; integration/e2e/coverage run it implicitly as needed
just test unit                    # Unit tests
just test integration             # Integration tests
just test e2e                     # Black-box e2e
just test coverage                # Coverage statistics
just test                         # Defaults to all, running unit / integration / e2e in sequence

# Packaging
just package check                # Read-only check of packaging prerequisites
just package bootstrap            # Fill in packaging prerequisites as needed (does not produce release artifacts)
just package wheel                # Build the online runtime package

# Quality
just qa                           # Show qa help
just qa precommit                 # Run the full pre-commit gate
just qa pip-audit                 # Dependency vulnerability audit
```

### 2.4. Development notes

- The Python required to run the project does not depend on the version preinstalled on the machine; `just prep sync` downloads managed Python 3.14 via `uv`,
  and installs the dependencies into the current project's `.venv/`.
- `just prep certs` prepares the server certificate, trust anchor and health-check client certificate from the shared development PKI.
- `just dev start` automatically completes environment preparation before starting (you can also run `just dev bootstrap` separately to only warm up without starting).
- `9007` is only for Leaders to call over mTLS, and `9008` is only for RabbitMQ to call; do not mix the two kinds of interfaces.
- The `tests/e2e/` in this repository starts a temporary dual-listener instance for black-box verification and does not depend on sibling services.
- Certificate mounting, stage prerequisites and upgrade procedures for production deployment are all documented in `acps-infra/README_en.md`.

## 3. Packaging and deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installer package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the installer package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses this repository's `just` / `scripts/`. Only when you need to manually produce this component's artifacts for the installation layer to collect should you use this repository's `just package wheel` (see `justfile`; not covered in detail here).
