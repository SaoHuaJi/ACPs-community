**[English](README_en.md) | [中文](README.md)**

# ACPs install-packaging

This tree is the ACPs **install layer** source: it consumes `image-packaging` / application release packages / vendor package artifacts, and uses Ansible to perform **image-mode** (Docker Compose) and **host-mode** (venv/systemd + OS packages + vendor tarball) first install and operations.

Step-by-step tutorials (recommended reading order):

- Assemble the install package → [`acps-docs/tutorials/install-package-build_en.md`](../../../acps-docs/tutorials/install-package-build_en.md)
- First-install deployment → [`acps-docs/tutorials/install-package-ansible-deploy_en.md`](../../../acps-docs/tutorials/install-package-ansible-deploy_en.md)
- Day-2 operations (renewal / trust / upgrade / rollback) → [`acps-docs/tutorials/install-package-day2-ops_en.md`](../../../acps-docs/tutorials/install-package-day2-ops_en.md)

The rest of this document covers the authoritative in-package switches and commands; the tutorials do not replace the detail tables in this README.

## Boundaries

| What this tree does | What this tree does not do |
| --- | --- |
| image-mode first-install orchestration (`acps_deploy_mode: image`) | Image builds (see `../image-packaging/`) |
| host-mode first-install orchestration (`acps_deploy_mode: host`; Rocky 8/9, Ubuntu 20.04/22.04) | Vendor package download pipeline (provide `vendor-bundle-dir` yourself at build time) |
| The ansible / templates / scripts needed to assemble an install package | Application release package assembly (see `../app-packaging/`) |
| Certificate provision / near-expiry renewal / trust-bundle refresh / basic smoke | CH/OS TLS (deferred); long-running renewal daemon |
| Version state registration; image / host component upgrade and artifact rollback | Automatic DB schema rollback; re-signing certificates on the default upgrade path; automatic os_package downgrade |

**No back-flow into packaging**: do not modify business wheel builds; do not write topology into image packages.

## Playbook entry points at a glance

| Purpose | Playbook | Notes |
| --- | --- | --- |
| First install | `playbooks/site.yml` | Full installation; **not** the day-2 upgrade entry point |
| Preflight | `playbooks/preflight.yml` | Cluster preflight |
| Near-expiry renewal | `playbooks/renew-certs.yml` | Renew leaf certificates according to the near-expiry window |
| trust-bundle refresh | `playbooks/refresh-trust-bundle.yml` | Distributes the trust chain; does not re-sign leaves |
| Version state backfill | `playbooks/register-state.yml` | Backfills component version state; no recreate |
| Component upgrade | `playbooks/upgrade.yml` | Shared by image / host; requires an explicit component list |
| Artifact rollback | `playbooks/rollback.yml` | Shared by image / host; switches back to `previous` |
| Smoke | `playbooks/smoke.yml` | **basic only** (product path) |
| Business acceptance | `playbooks/business.yml` | Optional after demo; fixed Steps A–D; requires both demos enabled |
| Discovery alive wait | `playbooks/wait-discovery-alive.yml` | **Optional**; per-AIC readiness gate after day2 / forced recreate and before `business.yml` (D2 only; default 480s + one more attempt after a settle on failure) |

### Prohibitions (upgrade / rollback)

- It is **forbidden** to run `docker compose down -v` (or any equivalent volume destruction): upgrade and rollback only use `up` / `up-recreate`, preserving data directories and named volumes.
- It is **forbidden** to use the full `site.yml` as an upgrade entry point; run first install again only when changing topology or doing a fresh installation.
- It is **forbidden** to bind certificate renewal into every upgrade by default; renewal / trust-chain refresh go through dedicated playbooks.
- **Do not** automatically `migrate downgrade`; rollback does not promise to undo forward migrations that have already been executed.

## Minimum tooling

- Ansible: `ansible-core` **≥ 2.16** (2.18.x recommended); installed on the **control node**
- **Control node** Python **3.11+** (`tomllib`): runs `scripts/manifest_lookup.py` and others; after unpacking, preferably run `scripts/bootstrap_control_ansible.sh` first (requires uv)
- **Target machines**: Docker Engine + Compose v2 plugin; a working system `python3` is enough (`resolve_image_uid.py` does **not** require 3.11 / `tomllib`)
- Repo development gate: `./scripts/syntax-check.sh` (the in-tree `.venv-tools` may be used; includes baseline ↔ image version alignment assertions)
- Version alignment only: `./scripts/assert-baseline-image-alignment.sh` (S12 subset: `--s12-only`)
- **Operations documentation index**: [`docs/README_en.md`](./docs/README_en.md); known limitations: [`docs/known-limitations.md`](./docs/known-limitations.md)

## Inventory examples

| File | Purpose |
| --- | --- |
| `inventories/hosts.example.yml` | image single machine |
| `inventories/hosts.rocky9.yml` / `hosts.ubuntu22.yml` / `hosts.rocky8.yml` / `hosts.ubuntu20.yml` | host single machine |
| `inventories/hosts.4os-multi.yml` + `host_vars/*.acps.local.yml` | host four-OS mixed deployment (one machine per group per component) |
| `inventories/hosts-multi.example.yml` | image multi-machine (app + deps) |
| `inventories/host_vars/*.example.yml` | multi-machine advertise / SSH examples |

## Assemble → unpack acceptance (closed loop)

**Recommended path**: official `image-packaging` flattened long name `*.image.tar.gz` + application release package directory → **single script** `build-install-package.sh`. The long name is kept inside the package; no renaming to a short name; do not pull/save demo-nginx / fluent-bit in this tree.

```bash
cd release/install-packaging

APP_RELEASE_DIR=/tmp/acps-app-release-output
IMAGE_OUT=/tmp/acps-image-packages

# macOS control node example: image linux-arm64, CLI darwin-arm64
./scripts/build-install-package.sh \
 --image-dir "$IMAGE_OUT" \
 --app-release-dir "$APP_RELEASE_DIR" \
 --image-platform linux-arm64 \
 --control-platform darwin-arm64 \
 --out-dir ./dist
# → dist/acps-image-install-{version}-{image_platform}.tar
```

For a remote amd64 business machine while the local machine still acts as the control node: `--image-platform linux-amd64`, and `--control-platform` still uses the local CLI (e.g. `darwin-arm64`).

Tutorials: [Assembling the install package (image / host)](../../../acps-docs/tutorials/install-package-build_en.md) · [Ansible deployment](../../../acps-docs/tutorials/install-package-ansible-deploy_en.md).

```bash
# 3) Simulate transfer to the control node: unpack into a separate directory (acceptance directly from the git source tree is forbidden)
rm -rf /tmp/acps-image-install-verify
mkdir -p /tmp/acps-image-install-verify
tar -xf ./dist/acps-image-install-*.tar -C /tmp/acps-image-install-verify
PKG=/tmp/acps-image-install-verify/acps-image-install-*-linux-arm64

# 4) Configure the inventory (single local machine: control node = business node)
cp -a "$PKG/ansible/inventories/hosts.example.yml" "$PKG/ansible/inventories/hosts.yml"
cp -a "$PKG/ansible/inventories/secrets.example.yml" "$PKG/ansible/inventories/secrets.yml"
# Edit secrets.yml; confirm host_vars/acps-node-1.yml, monitor_server_enabled=true, keycloak_enabled=false
# CA intermediate certificates: place them in inventories/ca-materials/ (ca.crt / ca.key / root-ca.crt)
# "$PKG/scripts/generate_ca_materials.sh" --out "$PKG/ansible/inventories/ca-materials/"

# 5) Before acceptance clear the pre-installed acps/*, forcing the in-package docker load
docker images --format '{{.Repository}}:{{.Tag}}' | grep '^acps/' | xargs -r docker image rm -f

# 6) Deploy using only the ansible + artifacts inside the unpacked directory
export ANSIBLE_CONFIG="$PKG/ansible/ansible.cfg"
cd "$PKG/ansible"
ansible-playbook playbooks/site.yml -i inventories/hosts.yml -e @inventories/secrets.yml
# Run once more to converge the final state (with no material/config changes there should be few recreates; do not treat site as upgrade)
# The log should contain docker load and no 「using local acps-cli source」
```

## host-mode first install (Rocky 8/9 · Ubuntu 20.04/22.04)

Tutorial entry points: [Assembling the install package §2](../../../acps-docs/tutorials/install-package-build_en.md) · [Ansible deployment (〔host〕 branch)](../../../acps-docs/tutorials/install-package-ansible-deploy_en.md).

### Topology

| Role | Host | Notes |
| --- | --- | --- |
| Build + Ansible control node | **local macOS** | Builds the host install package and runs `ansible-playbook`; is **not** a business node |
| Business node A | `rhel9.acps.local` | Rocky Linux 9 / RHEL 9 family; a single machine hosts all components |
| Business node B | `ubuntu22.acps.local` | Ubuntu 22.04; a single machine hosts all components |
| Business node C | `rhel8.acps.local` | Rocky Linux 8 / RHEL 8 family; a single machine hosts all components |
| Business node D | `ubuntu20.acps.local` | Ubuntu 20.04; a single machine hosts all components |

Business nodes must be reachable from the control node via **passwordless SSH + passwordless sudo**. Inventory examples:

- `ansible/inventories/hosts.rocky9.yml` / `hosts.ubuntu22.yml`
- `ansible/inventories/hosts.rocky8.yml` / `hosts.ubuntu20.yml` (a separate `acps_control_root` must be set)
- `ansible/inventories/hosts.4os-multi.yml` (four-OS mixed deployment; accompanied by `host_vars/*.acps.local.yml`, with python3.8 written only in the rhel8 host_vars)

(Copy to `hosts.yml` and replace the SSH user; for `secrets.yml` see `secrets.example.yml`.)

**Four-OS acceptance**: full acceptance of the host path must pass on **each of rocky8 / rocky9 / ubuntu20 / ubuntu22**; when extending to a new OS, at minimum the new OS must be green as a single machine, and the existing Rocky9/Ubuntu22 default artifacts (jammy fluent-bit and the like) must change by zero.

### Prerequisites

| Category | Requirement |
| --- | --- |
| **Control node** | Ansible `ansible-core` ≥ 2.16; Python 3.11+ (`tomllib`); `scripts/bootstrap_control_ansible.sh` |
| **Business node OS** | Rocky Linux 8/9 or Ubuntu 20.04/22.04 (`acps_os_id` ∈ baseline-matrix `[os_whitelist]`); systemd; distributions outside the whitelist are unsupported. **Rocky 8** requires **Python ≥3.8** preinstalled (`python38` module; `hosts.rocky8.yml` already sets `ansible_python_interpreter: /usr/bin/python3.8`), because ansible-core ≥2.16 does not support managed-node Python 3.6 |
| **OS online repositories** | PG (PGDG + pgvector; **Ubuntu 20 uses apt-archive**, live focal is already offline), Redis ≥ 7 (Ubuntu: `packages.redis.io`; **Rocky 8: Remi `redis:remi-7.2`**; Rocky 9: AppStream `redis:7`), RabbitMQ (Team RabbitMQ repository; **Ubuntu 20 pins `rabbitmq-server=4.2.8-1`**, because focal erlang maxes out at 26.x); `tools/` **cannot** substitute for apt/dnf |
| **Vendor tarball** | Build machine default cache `release/install-packaging/.vendor-bundle/`; `build-install-package.sh --mode host` **automatically downloads missing items** according to the url/sha256 in `baseline-matrix.toml` (you may pre-place them manually to speed things up; `--vendor-offline` uses the cache only). Includes **Temurin JRE17** (Keycloak `JAVA_HOME`). `amp_forwarder` is a dual artifact: jammy (rocky9/ubuntu22) + bionic/glibc228 (rocky8/ubuntu20) |
| **Java (Keycloak)** | host: the **Temurin JRE17** vendor inside the install package (`[vendor.temurin_jre17]`), the role writes `JAVA_HOME`, and OpenJDK OS packages are **no longer** installed online. OpenSearch still uses the JDK bundled with the official tarball. In image mode the Keycloak image ships its own JRE |
| **Python offline (optional)** | `--bundle-python-dir` packs into `tools/` (pinned uv + CPython); otherwise the role installs online according to `baseline-matrix [python]` |
| **Egress MTU (LLM/embedding)** | The host business NIC is recommended to use **MTU ≤1400** (a site network item). image-mode: the shared network `acps-net` is set to `acps_docker_mtu` (default **1400**); **Linux Engine** additionally merges mtu into `/etc/docker/daemon.json`; **Docker Desktop** skips daemon.json. A network that already has a wrong MTU needs `-e acps_docker_network_recreate=true` to be recreated. You can self-check with `ping -M do -s 1372 <gateway-ip>` |

preflight validates `mode ∈ {image,host}`, the OS whitelist (`acps_os_id`), OS repository reachability (`preflight_host_os_repos.yml`), the early vendor glibc gate on rocky8/ubuntu20, and vendor file existence (inside the install package).

You must rebuild the install package by running `build-install-package.sh --mode host` from the **source tree** before verifying again; it is forbidden to change only the ansible/matrix in the unpacked tree while reusing the old `artifacts/vendor`.

### Assembling the host install package

```bash
cd release/install-packaging

APP_RELEASE_DIR=/tmp/acps-app-release-output

# Default cache .vendor-bundle/: if missing, download per baseline-matrix url and verify sha256
./scripts/build-install-package.sh \
  --mode host \
  --app-release-dir "$APP_RELEASE_DIR" \
  --target-platform linux-amd64 \
  --control-platform darwin-arm64 \
  --out-dir ./dist
# → dist/acps-host-install-{version}-linux-amd64.tar

# Optional: explicit cache directory / air-gap
#   --vendor-bundle-dir /path/to/cache
#   --vendor-offline

# You can also pre-fetch separately:
# python3 scripts/ensure_vendor_bundle.py --matrix baseline-matrix.toml --arch amd64 --cache-dir .vendor-bundle
```

The artifact contains `artifacts/apps|control|vendor/`, `baseline-matrix.toml`, the build-time `release-manifest.toml` (including `artifact_kind`), `ansible/`, `templates/`, `scripts/`; there is **no** `images/` and **no** `bin/acps-install`.

`baseline-matrix.toml` pins `url` (or `url_amd64`/`url_arm64`) and `sha256_amd64`/`sha256_arm64` (or a unified `sha256`) for each `[vendor.*]`; some components use `fetch=` for post-download transformation (`clickhouse_flat` / `minio_wrap` / `fluentbit_deb`). `amp_forwarder` can additionally pin `file_glibc228` / `url_glibc228` / `sha256_*_glibc228` (selection for rocky8/ubuntu20).

### Unpack → multi-OS deployment (illustrative)

```bash
rm -rf /tmp/acps-host-install-verify
mkdir -p /tmp/acps-host-install-verify
tar -xf ./dist/acps-host-install-*.tar -C /tmp/acps-host-install-verify
PKG=/tmp/acps-host-install-verify/acps-host-install-*-linux-amd64

cp -a "$PKG/ansible/inventories/secrets.example.yml" "$PKG/ansible/inventories/secrets.yml"
# Edit secrets.yml; put CA materials in inventories/ca-materials/

export ANSIBLE_CONFIG="$PKG/ansible/ansible.cfg"
cd "$PKG/ansible"

# Connectivity (run once per business machine)
ansible -i inventories/hosts.rocky9.yml all -m ping
ansible -i inventories/hosts.ubuntu22.yml all -m ping
ansible -i inventories/hosts.rocky8.yml all -m ping
ansible -i inventories/hosts.ubuntu20.yml all -m ping

# First install (must be executed separately per OS; the following is the Rocky 9 example)
ansible-playbook -i inventories/hosts.rocky9.yml playbooks/preflight.yml -e @inventories/secrets.yml
ansible-playbook -i inventories/hosts.rocky9.yml playbooks/site.yml -e @inventories/secrets.yml
ansible-playbook -i inventories/hosts.rocky9.yml playbooks/smoke.yml -e @inventories/secrets.yml
# Product path (after both demos + LLM + full AMP): business.yml

# Other OSes: swap in hosts.ubuntu22.yml / hosts.rocky8.yml / hosts.ubuntu20.yml and repeat the commands above
```

The canonical working directory is **`ansible/`** inside the unpacked package; the entry point is `ansible-playbook playbooks/site.yml` (**not** `bin/acps-install`).

### host-mode capability summary

- **Hosting**: application venv/systemd, OS-package PostgreSQL/Redis/RabbitMQ, Keycloak/AMP vendor, and the demo host path are all landed by Ansible roles.
- **Four OSes**: Rocky 8/9 + Ubuntu 20.04/22.04 each support `preflight` / `site` / `smoke` / `business.yml` (A–D). Ubuntu 20 PG depends on apt-archive (no ongoing security updates); Rocky 8 Redis goes through Remi.
- **Business acceptance knobs**: `secrets.example.yml` contains `discovery_skip_cpu_llm` / `embedding_timeout` / `acps_llm6_force_fallback` / `partner_force_accept_decision`; Step A skips `run-sync` by default (`ACPS_FORCE_DISCOVERY_SYNC=1` forces synchronization).
- **image-mode demo Web**: uses `demo-leader-web` (no separate demo-nginx); the acceptance order is `site.yml` → `smoke.yml` → `business.yml` A–D, and you can check `http://<host>:9030/api/v1/health`.
- **Upgrade / rollback**: `upgrade.yml` / `rollback.yml` support an explicit component list (including applications and some vendor); the host path is usable on **all four OSes** (rocky8/9, ubuntu20/22). ClickHouse / OpenSearch are currently plaintext HTTP. For vendor downloads see `scripts/ensure_vendor_bundle.py`.
- **Multi-OS serial caveat**: the `~/.local/share/acps/control/work/certs` leaf marker of the same controller is reused across business machines; after changing OS/hostname (advertise / SAN), `cert_provision` automatically re-signs by fingerprint. Issuer changes likewise clear and re-sign automatically. After certificates/trust are landed, restart the direct consumers and their dependency closure. `smoke_basic` / renew / refresh perform fail-closed handshake acceptance for Redis TLS, AMQPS, mq-auth mTLS, registry:9002, and demo HTTPS. Each OS topology must use a separate `acps_control_root` (see `hosts.rocky8.yml` / `hosts.ubuntu20.yml`).
- **Do not** globally pass `-e acps_python_bin=/opt/...` to a mixed localhost+remote inventory (it would pollute the control node's CLI installation); rely on the per-host fact produced by `install_python`.

## Quick local development verification (not the closed-loop gate)

For orchestration debugging only, you may run inside the source tree (`acps_cli_allow_source_fallback=true` is allowed), but this **cannot** replace install-package closed-loop acceptance:

```bash
cd release/install-packaging
cp -a ansible/inventories/hosts.example.yml ansible/inventories/hosts.yml
cp -a ansible/inventories/secrets.example.yml ansible/inventories/secrets.yml
./scripts/syntax-check.sh
```

## Near-expiry renewal

After first install completes, scan leaf certificates against the near-expiry window and renew/distribute them (skipped when not near expiry and not forced). **Renewal keeps the Agent AIC unchanged** (same AIC `cert renew`; it is forbidden to delete+recreate a registered Agent because of force). **The scan does not cover advertise/SAN drift**; to fix the SAN go through `site.yml` (the AIC may change), and do not claim "renew has fixed the SAN".

```bash
cd "$PKG/ansible" # local development may also use release/install-packaging/ansible
ansible-playbook -i inventories/hosts.yml playbooks/renew-certs.yml \
 -e @inventories/secrets.yml
# Force re-signing leaves (still keeps AIC): -e acps_force_cert_renew=true
# Only some profiles: -e '{"acps_cert_renew_profiles":["registry-9002"]}'
```

## trust-bundle refresh

Pull the latest trust material from `ca_server` in the inventory and distribute it (by profile filename: `trust-bundle.pem` / `acps-root-ca.pem`). It does **not** re-sign leaf certificates:

```bash
cd "$PKG/ansible"
ansible-playbook -i inventories/hosts.yml playbooks/refresh-trust-bundle.yml \
 -e @inventories/secrets.yml
# Force overwrite: -e acps_force_trust_bundle_refresh=true
# Only some consumers: -e '{"acps_trust_bundle_profiles":["registry-9002","redis"]}'
```

## Version state registration

The successful first-install path writes `current` in `deploy_compose_service` / `control_acps_cli`. For an existing stack it can be backfilled (no recreate):

```bash
cd "$PKG/ansible"
ansible-playbook -i inventories/hosts.yml playbooks/register-state.yml \
 -e @inventories/secrets.yml
# Subset: -e '{"acps_register_components":["registry_server","discovery_server_cpu"]}'
```

Paths: `{{ acps_runtime_root }}/state/components/<component>.json`, `…/state/control/acps_cli.json`.

## Component upgrade

A dedicated entry point (**not** running the full `site.yml` again). Upgrades artifacts and processes by component; preserves data directories and certificates; Compose only `up` / `up-recreate` (**`down -v` is forbidden**). By default it does **not** invoke leaf renewal. When the CSV contains `ca_server`, the target contract is a follow trust refresh by default (`acps_upgrade_follow_refresh_trust`; see `docs/known-limitations.md`). **After a successful upgrade/rollback do not overwrite it with `site.yml`.**

```bash
cd "$PKG/ansible" # local development may also use release/install-packaging/ansible
ansible-playbook -i inventories/hosts.yml playbooks/upgrade.yml \
 -e @inventories/secrets.yml \
 -e acps_upgrade_components=registry_server
# Multiple components (order aligned with the C phase dependencies): -e acps_upgrade_components=registry_server,ca_server
# Skip post-upgrade smoke (not recommended): -e acps_upgrade_skip_smoke=true
# discovery has the alias discovery_server (expands to discovery_server_{{ variant }})
```

`acps_upgrade_components` is **required** (explicit is safer). Flow: preflight → `save_previous_state` → stage/load → render → migrate (in-role alembic / hook placeholder) → recreate → health/smoke → `commit_state`.

### host-mode upgrade / rollback (S5 finalized)

With `acps_deploy_mode: host` it is the same entry point; applications land in `releases/<ver>/` + a `current` symlink, and the unit goes through `current`. The first upgrade migrates the flat layout into that structure.

```bash
# Business machine (example ubuntu22 / rocky9 inventory)
ansible-playbook -i inventories/hosts.ubuntu22.yml playbooks/upgrade.yml \
  -e @inventories/secrets.yml \
  -e acps_upgrade_components=registry_server,ca_server
# When CLI is included, CLI comes before business apps:
# -e acps_upgrade_components=acps_cli,registry_server
# Force reinstall of the same artifact: -e acps_force_app_reinstall=true

ansible-playbook -i inventories/hosts.ubuntu22.yml playbooks/rollback.yml \
  -e @inventories/secrets.yml \
  -e acps_rollback_components=registry_server,ca_server
# If state.current.migrate_id is newer than previous: explicit acknowledgement is required (no schema downgrade)
# -e acps_rollback_acknowledge_migrate=true
```

host key points: if `artifact_id` is unchanged and not forced → no-op (health can still run); a failed migrate does not switch `current`; rollback only switches the artifact pointer and does **not** execute an alembic downgrade.

**vendor_bundle (host)**: `keycloak` (depends on `temurin_jre17` in the same package, does not enter the CSV separately) / `redpanda` / `victoria_metrics` / `clickhouse` / `minio` / `opensearch` / `amp_forwarder` can be selected in the component CSV of `upgrade.yml` / `rollback.yml`; the switch goes through `releases/`+`current` and does **not** clear `acps_data_root`. `amp_forwarder` is also first-installed with the `site.yml` Forwarder phase (tag `phase_14_amp_forwarder`); with the same artifact upgrade is a no-op by default (no `previous` written), and for a rollback drill you can use `-e acps_force_vendor_reinstall=true` to retain `previous`. See `docs/known-limitations.md` for details.

**os_package (PostgreSQL / Redis / RabbitMQ)**:

- Upgrade: `ensure_repos →` (PG) **major-version gate** → `package state=latest` (same major, minor version) → idempotent config rendering → restart → health; state records the **package version string**, and there is **no** `releases/`.
- PostgreSQL: the installed cluster major (`PG_VERSION`) must equal `postgresql_os_major_version` (baseline `17`); drift → **fail fast**, prompting manual `pg_upgrade` / dump-restore; the data directory is **not** changed.
- Rollback: for `postgresql` / `redis` / `rabbitmq` it is **refused by default** (no automatic `dnf`/`apt` package downgrade). Remove these components from `acps_rollback_components`, or restore the package set manually.
- Negative gate (runnable locally): `./scripts/h4_s4_pg_major_neg_check.sh` (log `/tmp/acps-h4-s4-pg-major-neg.log`).

```bash
# Example: upgrade PG to a minor version within the same major (host first install must already be done)
ansible-playbook -i inventories/hosts.ubuntu22.yml playbooks/upgrade.yml \
  -e @inventories/secrets.yml \
  -e acps_upgrade_components=postgresql
# The following will fail (the product refuses automatic package downgrade):
# -e acps_rollback_components=postgresql
```

## Artifact rollback

A dedicated entry point (**not** `site.yml` / nor an automatic chained rollback). It switches the running pointer of the selected components back to the `previous` in state (or an explicit `acps_rollback_to` which must match that previous); it does **not** automatically roll back the DB/schema; it does **not** include certificates in the default rollback set; Compose only `up` / `up-recreate` (**`down -v` is forbidden**). On failure it stops and preserves the scene. **After rollback do not overwrite it with `site.yml`** (this easily breaks the rollback intent).

```bash
cd "$PKG/ansible" # local development may also use release/install-packaging/ansible
ansible-playbook -i inventories/hosts.yml playbooks/rollback.yml \
 -e @inventories/secrets.yml \
 -e acps_rollback_components=registry_server
# Optional target (must equal the retained previous.version or previous.image_tag):
# -e acps_rollback_to=2.2.0
# Skip post-rollback smoke (not recommended): -e acps_rollback_skip_smoke=true
```

`acps_rollback_components` is **required**. Flow: preflight → read state → validate previous / image availability (local tag or in-package artifact) → reuse the component role (pin `acps_image_tag`) → recreate → health/smoke → `commit_state` (`current=restored`, `previous` cleared). A rollback after an upgrade with the same tag can still exercise the full path (pointer switch + state update), but it does not constitute cross-version artifact evidence.

## AMP Forwarder

After the Demo is deployed, the install layer uses **Fluent Bit** (`roles/amp_forwarder`) to tail the six kinds of `amp_*.jsonl` of Leader/Partner and forward them to Redpanda's `amp.audit` / `amp.access` / `amp.message` / `amp.metrics` / `amp.heartbeat` / `amp.system`. Application processes do not connect to Kafka directly.

| Item | Convention |
| --- | --- |
| Form | **Standalone compose service** + **host shared log directory** (not a demo sidecar). The demo compose binds `/opt/acps/app/logs` to `{{ acps_log_root }}/demo_{leader,partner}`; the Forwarder mounts the same directory read-only. |
| When to deploy | The AMP Forwarder phase in `site.yml`; tag `phase_14_amp_forwarder` (after the demo phase). When both demos are off the role is a no-op. |
| Broker | Derived from the inventory: same machine as Redpanda uses `redpanda:9092`, cross-machine uses `acps_group_addr(redpanda):redpanda_kafka_port`. |
| Image | `acps/fluent-bit:…` from image-packaging (manifest `[images.fluent_bit]`). It must be packed into the image package before assembling the install package. |
| Kafka topics | `roles/redpanda` idempotently pre-creates the six `amp.*` kinds + DLQ + `amp.heartbeat.alive-delta` after up (partitions / `LogAppendTime` aligned with `dev-infra.sh`). For non-audit the Forwarder disables auto-create. |
| Monitor Writers | The `monitor_server` production overlay (`production.toml.j2`) points at the installed Redpanda; message/system `writer_enabled=true`; the heartbeat inbound partitions match topic creation. |
| Monitor→Discovery heartbeat sync | See the next section (Relay + alive-sync). |

```bash
# Deploy / re-run only the Forwarder (demo already enabled and the log directory already exists)
cd "$PKG/ansible"
ansible-playbook -i inventories/hosts.yml playbooks/site.yml \
 -e @inventories/secrets.yml --tags phase_14_amp_forwarder
```

Until the Forwarder + topics + Writers + heartbeat sync are all in place, you must not claim business acceptance has passed.

## Monitor→Discovery heartbeat sync

The install layer **explicitly** configures and enables the Monitor Heartbeat Relay and Discovery alive-sync (rendered into TOML from the inventory; do **not** hand-edit configuration inside containers as the main path).

| Item | Convention |
| --- | --- |
| Monitor Relay | `roles/monitor_server/templates/production.toml.j2`: `[heartbeat] sync_enabled=true`, `delta_topic=amp.heartbeat.alive-delta`, sharding/inbound partitions consistent with topic creation; Kafka bootstrap same machine `redpanda:9092` / cross machine `acps_group_addr(redpanda):redpanda_kafka_port`. |
| Discovery alive-sync | `roles/discovery_server/templates/production.toml.j2`: when `monitor_server_enabled=true`, `enabled=true` + `auto_start=true`; `provider_base_url` → Monitor `/acps-amp-v1/heartbeat` (same-machine Compose DNS `monitor_server:9009` / cross-machine advertise); `kafka_bootstrap_servers` / `kafka_topic=amp.heartbeat.alive-delta` / `kafka_group_id=discovery-server.alive-sync.v1`. |
| TLS / authentication | Kafka **PLAINTEXT** (consistent with Writers/Forwarder). Bootstrap goes through Monitor `/sync/info` + `/sync/snapshot`: a shared service-to-service Bearer (`secrets.yml` → `monitor_heartbeat_sync_internal_token` → Monitor `HEARTBEAT_SYNC_INTERNAL_TOKEN` + Discovery `ALIVE_SYNC_PROVIDER_BEARER_TOKEN`). **Mandatory when Keycloak OIDC is on** (Monitor `/sync/*` would otherwise require an operator JWT); when OIDC is off it still works without a Bearer, but the install layer still requires that secret (roles assert). A human operator may also call `/sync/*` with an OIDC Bearer. |
| Business acceptance matrix | When demos are enabled (`demo_leader` + `demo_partner`) and business acceptance is to be run, you must simultaneously enable **Monitor + Discovery + this sync chain** (plus Forwarder / topics / Writers). **Keycloak is supported** (shared sync token). If any is missing you must not claim business acceptance has passed. |
| D2 pass condition | Check heartbeat/liveness **only** in Discovery (`aliveCount` of `/admin/alive-sync/status` / discover `aliveMap`). It is **forbidden** to use the Monitor heartbeat summary / Query API as the D2 pass condition. Failure messages point to the **Monitor Relay** or the **Discovery alive-sync consumer**. |

```bash
# Gates (illustrative; when the control plane is reachable)
# 1) Monitor Provider (not D2; only confirms the Relay/Sync Profile)
# Keycloak/OIDC on: must carry the shared sync Bearer (consistent with secrets.yml)
curl -s -H "Authorization: Bearer <monitor_heartbeat_sync_internal_token>" \
 "http://<monitor>:9009/acps-amp-v1/heartbeat/sync/info" | python3 -m json.tool
# Expect type=amp-alive-delta kafkaTopic=amp.heartbeat.alive-delta

# 2) Discovery consumer (the D2 path)
curl -s "http://<discovery>:9005/admin/alive-sync/status" | python3 -m json.tool
# Expect running=true and checkpointCount>=1; after a demo+Forwarder heartbeat cycle aliveCount>=1 (full Leader+Partner belongs to )

# The basic smoke test automatically probes the Sync Profile + running when both Monitor+Discovery are enabled (Bearer injected from secrets)
# After demo is enabled and the Forwarder has run, the alive-sync gate of site.yml requires aliveCount>=1
ansible-playbook -i inventories/hosts.yml playbooks/smoke.yml -e @inventories/secrets.yml
```

## LLM / secrets injection

Keys are written in `inventories/secrets.yml` (**do not commit to git**; for an example see `secrets.example.yml`, placeholders only). Roles render them into each component's env:

| Purpose | secrets key | Render target | Notes |
| --- | --- | --- | --- |
| Discovery (CPU) | `discovery_llm_*`, `embedding_*` | `discovery_server` env | `discovery_server_variant: cpu` (default) usually needs real keys before NL queries can hit |
| Discovery (GPU) | the same keys may still be rendered | same as above | `discovery_server_variant: gpu` usually does **not** depend on an external LLM; keys may be left as placeholders / unused |
| Demo Leader/Partner | `demo_llm_*` (shared default); optional `demo_leader_llm_*` / `demo_partner_llm_*` per tier | `LEADER_LLM_*` / `PARTNER_LLM_*` | when the per-tier keys are unset it falls back to `demo_llm_*` |

**Business acceptance contract (`business.yml` Step A)**:

- It is **forbidden** to assert "whether the Discovery LLM secrets exist" alone as a pass/fail condition.
- Step A **only** requires a natural-language query to hit for each demo Partner; missing or wrong keys manifest as query misses and therefore failure.
- Step B/C dialogue depends on Leader/Partner LLM; not configuring it causes business assertions to fail (likewise not a standalone secrets-existence probe).

Operations tip: for CPU Discovery inject usable `discovery_llm_*` / `embedding_*` in `secrets.yml`; the GPU variant may skip this.

## Business acceptance (`business.yml`)

Recommended order:

1. `site.yml` (including demo + AMP Forwarder + alive-sync gate)
2. `smoke.yml` (**basic only**)
3. (Optional) `business.yml` — runs the fixed Steps A→B→C→D once, to completion
4. When running business again after day2 / a forced recreate: first `wait-discovery-alive.yml` (D2 only; after force renew the aliveMap may lag, default 480s timeout and after a failure settle 90s then retry once), then `business.yml`

**Business acceptance cannot be run without demos**: `demo_partner_enabled=true` **and** `demo_leader_enabled=true` are required; otherwise `business.yml` fails the assert at the beginning (a successful install still depends only on the basic smoke test).

```bash
cd "$PKG/ansible"
ansible-playbook -i inventories/hosts.yml playbooks/site.yml \
 -e @inventories/secrets.yml
ansible-playbook -i inventories/hosts.yml playbooks/smoke.yml \
 -e @inventories/secrets.yml
# Optional: after both demos are enabled and LLM / Forwarder / Writers / heartbeat sync are configured
ansible-playbook -i inventories/hosts.yml playbooks/business.yml \
 -e @inventories/secrets.yml
# Expected: exit 0; A–D all green
```

Conventions:

- **Business acceptance is not mandatory at first install**: `site.yml` does not run `business.yml`; after installation you may execute it as needed.
- **The product path = one whole pipeline run**: there is no product parameter to "run only a certain segment" (no `acps_business_tests=…`); the internal `business_step_*` tags are for debugging only.
- **Business acceptance entry point**: use `business.yml` (do not use the deprecated `acps_smoke_kind=business` / `biz_*` probe groups).
- **Each repository keeps its own e2e**: install-layer business acceptance does not replace each application repository's `tests/e2e`; the two are complementary.

## Known limitations

- The gates and the upgrade→rollback closed loop are mostly **same tag**: they validate the path and the state pointer, **not** cross-version artifact evidence.
- The local closed loop disables Monitor / OIDC / demo by default (they can be enabled via the inventory).
- **host-mode**: `upgrade.yml` / `rollback.yml` support four OSes (rocky8/9, ubuntu20/22); vendor is cached and downloaded by `ensure_vendor_bundle` per matrix url/sha256; if egress hits a PMTU black hole see the prerequisites MTU≤1400.
- **image-mode**: demo Web uses `demo-leader-web` (no demo-nginx).
- PostgreSQL **major-version** upgrade: the host path **fails fast** (`check_postgresql_os_major.yml`; installed `PG_VERSION` vs `postgresql_os_major_version` / baseline); a major version requires manual `pg_upgrade` / dump-restore. Negative gate: `./scripts/h4_s4_pg_major_neg_check.sh`.
- ClickHouse / OpenSearch are currently plaintext HTTP; certificate renewal goes through `renew-certs.yml` / `refresh-trust-bundle.yml` (no built-in long-running daemon).
- For step-by-step operations see [docs/README_en.md](docs/README_en.md) and [acps-docs](../../../acps-docs/README_en.md).
