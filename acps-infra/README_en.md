**[English](README_en.md) | [中文](README.md)**

# acps-infra

The infrastructure and **product delivery** repository for ACPs: shared local-development dependencies (`dev-infra`), plus unified **packaging / installation / upgrade / rollback** (`release/install-packaging`).

The main product path is:

```text
application release package (app-release)
    ├─→ [image] image package → acps-image-install-*.tar
    └─→ [host]  vendor  → acps-host-install-*.tar
                              │
                              ▼
                    Ansible (site / upgrade / rollback / business)
```

For concepts see below; **for concrete commands and step-by-step procedures always see the [acps-docs tutorials](../acps-docs/README_en.md)**; this README does not repeat long command lists.

## 1. Overview

### 1.1. What this repository is responsible for

| Directory | Responsibility |
| --- | --- |
| `dev-infra/` | Shared local-development dependencies (Postgres / Redis / RabbitMQ, etc.) |
| `release/install-packaging/` | **Installation layer**: assemble the installation package + Ansible first install / certificates / smoke / business / upgrade and rollback |
| `release/app-packaging/` | Application release package assembly (collected by the installation layer) |
| `release/image-packaging/` | image-mode image packages (consumed by the image installation package) |
| `dev-infra/keycloak/` | Development integration Keycloak realm / bootstrap |

### 1.2. Key directories (delivery-related)

```text
acps-infra/
├── dev-infra/                      # shared local-development dependencies
├── release/
│   ├── app-packaging/              # application release package
│   ├── image-packaging/            # Docker image packages (image only)
│   └── install-packaging/          # installation package assembly + Ansible
│       ├── ansible/                # site / upgrade / rollback / business …
│       ├── docs/                   # installation layer short index
│       └── README.md               # installation layer details entry point
└── …
```

### 1.3. When to use which entry point

| Goal | Entry point |
| --- | --- |
| Start the local-development shared dependencies | `dev-infra/dev-infra.sh` (see §2) |
| Understand the installation layer boundary / playbook overview | [`release/install-packaging/README_en.md`](release/install-packaging/README_en.md) |
| **Assemble an installation package (step-by-step commands)** | [acps-docs: assembling an installation package](../acps-docs/tutorials/install-package-build_en.md) |
| **Ansible deployment / acceptance** | [acps-docs: Ansible deployment](../acps-docs/tutorials/install-package-ansible-deploy_en.md) |
| Multi-machine topology | [Three-node tutorial](../acps-docs/tutorials/install-package-ansible-deploy-3nodes_en.md) or `hosts-multi.example.yml` inside the package |
| **Day-2 operations (renewal / trust / upgrade / rollback)** | [acps-docs: day-2 operations](../acps-docs/tutorials/install-package-day2-ops_en.md) |
| Application release package / image packages | [application release package](../acps-docs/tutorials/app-release-package-build_en.md), [image packages](../acps-docs/tutorials/docker-image-packages-from-app-release_en.md) |

## 2. Development (shared dependencies)

### 2.1. Prerequisites

- [uv](https://docs.astral.sh/uv/getting-started/installation/), [just](https://just.systems/man/en/packages.html) (commonly used by sibling projects)
- [Docker](https://www.docker.com/products/docker-desktop/) (`dev-infra`; also needed on image-mode target machines)

### 2.2. Local-development shared dependencies

```bash
./dev-infra/dev-infra.sh check
./dev-infra/dev-infra.sh up                  # default postgres
./dev-infra/dev-infra.sh up redis rabbitmq   # on demand
./dev-infra/dev-infra.sh status
./dev-infra/dev-infra.sh down
```

See [dev-infra/README_en.md](dev-infra/README_en.md). `dev-infra` serves local development only and is **not** a production deployment entry point.

## 3. Packaging and deployment (main product path)

### 3.1. Concepts

- **Two deployment modes** (`acps_deploy_mode`)  
  - **image**: Docker Compose on the target machine; the installation package carries images + the control-plane CLI.  
  - **host**: Rocky 9 / Ubuntu 22.04 on the target machine; venv/systemd + OS packages + vendor tarball.
- **Artifact chain**: each business repository produces application artifacts → `app-packaging` / `image-packaging` → **a single installation package** → the control node unpacks it and then runs Ansible.
- **Orchestration entry points** (all in the installation package or in the source tree `release/install-packaging/ansible/playbooks/`)  
  - First install: `site.yml`  
  - Basic probe: `smoke.yml`  
  - Business acceptance: `business.yml` (after the demo is running on both sides, fixed A–D)  
  - Upgrade / rollback: `upgrade.yml` / `rollback.yml` (an explicit component list is required; `down -v` is **forbidden**)  
  - Certificates: `renew-certs.yml` / `refresh-trust-bundle.yml`
- **Control node vs business nodes**: Ansible and acps-cli run on the control node; business nodes run Compose or systemd according to the mode.

### 3.2. Where to find the commands

| Topic | Document |
| --- | --- |
| Quick start (choosing a path) | [acps-docs getting-started](../acps-docs/getting-started/README_en.md) |
| Assembling `acps-*-install-*.tar` | [install-package-build_en.md](../acps-docs/tutorials/install-package-build_en.md) |
| Unpacking, secrets, CA, `site.yml` / `business.yml` | [install-package-ansible-deploy_en.md](../acps-docs/tutorials/install-package-ansible-deploy_en.md) |
| Splitting across three business nodes | [install-package-ansible-deploy-3nodes_en.md](../acps-docs/tutorials/install-package-ansible-deploy-3nodes_en.md) |
| Renewal / trust / upgrade / rollback | [install-package-day2-ops_en.md](../acps-docs/tutorials/install-package-day2-ops_en.md) |
| Installation layer playbooks / variables / known limitations | [`release/install-packaging/README_en.md`](release/install-packaging/README_en.md), [`docs/`](release/install-packaging/docs/README_en.md) |

Each sibling repository (registry / ca / discovery / …) keeps **only one chapter** in its README pointing to this document; component-level `just package` is used only for artifacts collected by the installation layer and does not constitute an independent full-stack deployment entry point.
