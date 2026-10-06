**[English](README_en.md) | [中文](README.md)**

# registry-server

registry-server is the Agent registry center of ACPs, responsible for Agent registration, review, ATR / EAB
related capabilities, and management of the registration data required for DSP synchronization. This document explains the project positioning and day-to-day development; for full-stack packaging and deployment, see Chapter 3.

## 1. Overview

### 1.1. Project positioning

- Provide external APIs for Agent registration, query, review and file upload
- Provide registration and identity-side support for the ACPs ATR / EAB process
- Provide DSP synchronization source data for `discovery-server`

### 1.2. Project characteristics

- Dual-plane operation: `9001` public API, `9002` mTLS API
- Local development always uses "host process + `../acps-infra/dev-infra`"
- Real cross-service integration testing is centralized in `acps-cli/tests/e2e/`; tests in this repository focus only on their own boundaries

### 1.3. Directory overview

```text
registry-server/
├── app/                  # Business code
├── alembic/              # Database migrations
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
cd registry-server

# It is recommended to copy the template explicitly first and check the key settings (just prep env / just dev start will also generate it if missing).
cp .env.example .env
# Edit .env: confirm sensitive items such as the database, token and certificate paths

just dev start
```

Common addresses after startup:

- Public API: `http://localhost:9001`
- Public Docs: `http://localhost:9001/docs`
- mTLS API: `https://localhost:9002`

### 2.3. Common commands

```bash
# Help
just help                         # Print a command overview; running just alone also shows help

# Shared dependencies
just infra up postgres            # Start the shared dependencies required by registry-server
just infra status                 # Show shared dependency status

# Environment preparation
just prep env                     # Generate .env from .env.example if missing
just prep sync                    # Download managed Python 3.14 and sync dependencies into .venv/
just prep hooks                   # Install/update Git hooks
just prep certs                   # Prepare local mTLS development certificates
just prep migrate dev             # Migrate the development database
just prep migrate test            # Migrate the test database

# Development
just dev check                    # Read-only check of the development environment, certificates and key settings
just dev bootstrap                # Optional: only warm up the environment, without starting services
just dev start                    # Start the public + mTLS dual plane in the background
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
- Before starting, `just dev start` automatically prepares `.venv`, hooks, development database migrations and local certificates (you can also run `just dev bootstrap` separately to only warm up without starting).
- `9002` uses real TLS + mandatory client certificate verification by default.
- `REGISTRY_SERVER_INTERNAL_API_TOKEN` must stay consistent with `ca-server`, which matters especially during real integration testing.
- To verify the complete integration chain between `registry-server`, `ca-server` and `discovery-server`, go to
  `acps-cli/tests/e2e/`.

### 2.5. OIDC historical local user binding

When `registry-server` switches from local accounts to OIDC, if you want existing Agents to remain under their original `User.id`,
you need to first explicitly bind the historical local users to the corresponding OIDC principal. The repository provides a management command that defaults to dry-run:

```bash
uv run python -m app.account.oidc_user_link_migration \
  --mapping-file /path/to/oidc-user-links.json
```

The mapping file must be a JSON array, and every record must explicitly provide `username` or `user_id`; matching automatically by email alone is not supported:

```json
[
  {
    "username": "alice",
    "issuer": "https://keycloak.example/realms/acps-registry",
    "subject": "real-oidc-subject-from-idp",
    "expected_email": "alice@example.com"
  }
]
```

Usage recommendations:

- Run dry-run first and confirm `blocking_count = 0`.
- The database is only actually written when `--apply` is passed:

```bash
uv run python -m app.account.oidc_user_link_migration \
  --mapping-file /path/to/oidc-user-links.json \
  --apply \
  --report-file /tmp/registry-oidc-link-report.json
```

- The tool does not silently merge users by email; `expected_email` only serves as a manual verification condition to avoid incorrect bindings.
- The output report includes `principal_id` and `subject_hash`, and does not echo the raw subject.
- The tool only pre-binds `external_issuer` / `external_subject` / `external_principal_id`, and does not directly rewrite the historical `User.id`.

## 3. Packaging and deployment

Full-stack delivery has been unified into the **`acps-infra` installation layer**: application release package → (image) image package → installer package → Ansible first install / upgrade / rollback. This repository is no longer an independent standalone / per-repository wheel full-stack deployment entry point.

- Concepts and directory entry point: [`acps-infra/README_en.md`](../acps-infra/README_en.md)
- Installation layer details: [`acps-infra/release/install-packaging/README_en.md`](../acps-infra/release/install-packaging/README_en.md)
- Step-by-step commands (acps-docs):
  - [Assemble the installer package](../acps-docs/tutorials/install-package-build_en.md)
  - [Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md)
  - [Application release package build](../acps-docs/tutorials/app-release-package-build_en.md)

Development integration still uses this repository's `just` / `scripts/`. Only when you need to manually produce this component's artifacts for the installation layer to collect should you use this repository's `just package wheel` (see `justfile`; not covered in detail here).
