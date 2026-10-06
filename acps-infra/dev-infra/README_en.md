**[English](README_en.md) | [中文](README.md)**

# dev-infra

`dev-infra` is the set of local development dependencies shared by multiple ACPs projects. Day-to-day development manages the services in the underlying [compose.yml](./compose.yml) through [dev-infra.sh](./dev-infra.sh), rather than writing `docker compose` commands by hand.

## Goals

- Provide a unified entry point for shared dependencies of projects such as `registry-server`, `ca-server`, `discovery-server`, and `acps-cli`
- Expose only stable service names externally, without exposing Compose implementation details such as `profile`
- Host ports match each service's native listening port (for example Redis `6379`, PostgreSQL `5432`), **but 90xx is reserved for the application services**; services such as Kafka / MinIO / ClickHouse native TCP that fall in 90xx are instead mapped into the `19xxx` range
- Project-level `Justfile` delegates uniformly to `dev-infra.sh` through [just/infra.just](./just/infra.just)

## Public services

| Public service     | Compose service        | Container name         | Host ports                         | volume                       | Description |
| ------------------ | ---------------------- | ---------------------- | ---------------------------------- | ---------------------------- | ---- |
| `postgres`         | `dev-postgres`         | `dev-postgres`         | `5432`                             | `dev-infra_dev-pgdata`       | Default dependency, includes each project's development and test databases |
| `redis`            | `dev-redis`            | `dev-redis`            | `6379`                             | `dev-infra_dev-redisdata`    | Optional; logical database numbers are partitioned by project + environment (see below) |
| `rabbitmq`         | `dev-rabbitmq`         | `dev-rabbitmq`         | `5671` (TLS), `15672` (management) | `dev-infra_dev-mqdata`       | Optional; in the development environment there is only the TLS broker |
| `gateway`          | `dev-nginx`            | `dev-nginx`            | `80`                               | none                         | Development gateway |
| `keycloak`         | `dev-keycloak`         | `dev-keycloak`         | `9080`                             | `dev-infra_dev-keycloakdata` | Optional; real-user OIDC / Keycloak integration testing |
| `kafka`            | `dev-redpanda`         | `dev-redpanda`         | `19092`, `19644`                   | `dev-infra_dev-redpandadata` | Host `19092` (avoiding the application 90xx range); inside the container network `dev-redpanda:9092` |
| `victoria-metrics` | `dev-victoria-metrics` | `dev-victoria-metrics` | `8428`                             | `dev-infra_dev-vmdata`       | Time-series database |
| `clickhouse`       | `dev-clickhouse`       | `dev-clickhouse`       | `8123` (HTTP), `19010` (native TCP) | `dev-infra_dev-chdata`       | Native TCP is mapped to `19010`, avoiding the application 90xx range |
| `minio`            | `dev-minio`            | `dev-minio`            | `19000` (API), `19001` (Console)   | `dev-infra_dev-miniodata`    | Avoids application ports such as registry `9001` |
| `opensearch`       | `dev-opensearch`       | `dev-opensearch`       | `9200`                             | `dev-infra_dev-opensearchdata` | System full-text search |

Legacy names are still supported:

- `dev-postgres`
- `dev-redis`
- `dev-rabbitmq`
- `dev-nginx`

The script still accepts the old names, but prints a deprecation notice; new documentation and project-level entry points uniformly use `postgres`, `redis`, `rabbitmq`, and `gateway`.

## Quick start

```bash
# Check Docker / Compose / compose.yml / service mappings
./dev-infra.sh check

# Start the default dependency (postgres)
./dev-infra.sh up

# Show the status of all services
./dev-infra.sh status

# Start additional dependencies
./dev-infra.sh up redis rabbitmq

# Start Keycloak (automatically completes realm / EdDSA / registry-cli, monitor-cli, registry-e2e, monitor-e2e, leader-e2e client bootstrap)
./dev-infra.sh up keycloak

# Start the dependencies required by monitor-server Access
./dev-infra.sh up redis kafka clickhouse

# Start the full monitor-server Message pipeline (including Writer consumption)
./dev-infra.sh up redis kafka clickhouse

# Wait until ready
./dev-infra.sh wait postgres rabbitmq

# View logs
./dev-infra.sh logs postgres rabbitmq --follow

# Stop the entire dev-infra
./dev-infra.sh down
```

## Command reference

### `check`

Check the runtime prerequisites:

- whether `docker` is available
- whether `docker compose` is available
- whether `compose.yml` is parseable
- whether the top-level project name matches the script constant
- whether the service and volume mappings are complete
- whether the external network `acps-dev-net` exists

Example:

```bash
./dev-infra.sh check
```

### `up [service ...]`

Start the specified services; when no service is passed, `postgres` is started by default.

Example:

```bash
./dev-infra.sh up
./dev-infra.sh up postgres
./dev-infra.sh up postgres rabbitmq
```

Notes:

- The first start automatically creates the external network `acps-dev-net`
- `up` only submits the start command; when you need to wait for health checks, run `wait` afterwards

### `down`

Stop the entire `dev-infra` compose project, keeping volumes.

Example:

```bash
./dev-infra.sh down
```

Notes:

- This is a whole-shutdown operation for the shared dependencies and affects all local projects currently using `dev-infra`
- Volumes are not deleted by default, so databases and message data are not cleared

### `status [service ...]`

Output static definitions and dynamic status.

Example:

```bash
./dev-infra.sh status
./dev-infra.sh status postgres rabbitmq
```

The output includes:

- public service name
- Compose service name
- container name
- port mappings
- volume name
- current status and health status
- service description

If the Docker daemon is currently unreachable, `status` degrades to a static view and marks the dynamic fields as `unavailable`.

### `wait [service ...]`

Wait for services to become ready.

Example:

```bash
./dev-infra.sh wait
./dev-infra.sh wait postgres
./dev-infra.sh wait postgres rabbitmq
```

Notes:

- When no service is passed, it waits by default for the currently created service containers
- For services with a healthcheck, it waits for `healthy`
- For services without a healthcheck, it waits for `running`

### `logs [service ...] [--tail N] [--since DURATION] [--follow]`

View logs, supporting a single service, multiple services, and follow mode.

Example:

```bash
./dev-infra.sh logs
./dev-infra.sh logs postgres
./dev-infra.sh logs postgres rabbitmq --tail 300
./dev-infra.sh logs rabbitmq --since 10m --follow
```

Notes:

- Outputs the last `200` lines by default
- Does not block by default; only `--follow` keeps following
- When no service is passed, it outputs only the logs of currently running services by default
- If you need to view the logs of a stopped container, specify the service explicitly

### `reset [service ...] [--volumes] [--yes]`

Perform a repair rebuild or an explicit data cleanup.

Example:

```bash
./dev-infra.sh reset postgres
./dev-infra.sh reset postgres --volumes --yes
./dev-infra.sh reset --volumes --yes
```

Notes:

- Without `--volumes`: delete the container and rebuild it, keeping the data
- With `--volumes`: delete the corresponding volume; the data is recreated on the next `up`
- A full `reset`, or any operation with `--volumes`, requires `--yes` to be passed explicitly
- `gateway` has no volume, so running `reset gateway --volumes --yes` only deletes the container and does not delete a data volume

## Databases

After `postgres` starts, it initializes the development and test databases through [postgres/init/01-create-databases.sh](./postgres/init/01-create-databases.sh).

To avoid building the shared `dev-postgres` on a third-party prebuilt image such as `pgvector/pgvector:pg17`, it is now built from the local [postgres/Dockerfile](./postgres/Dockerfile): the base image uses the official `postgres:17-bookworm`, and `postgresql-17-pgvector` is installed through Debian packages.

| Database               | User        | Password    | Purpose                 |
| ---------------------- | ----------- | ----------- | ----------------------- |
| `agent_registry`       | `registry`  | `registry`  | registry-server dev database |
| `agent_registry_test`  | `registry`  | `registry`  | registry-server test database |
| `agent_ca`             | `ca`        | `ca`        | ca-server dev database  |
| `agent_ca_test`        | `ca`        | `ca`        | ca-server test database |
| `agent_discovery`      | `discovery` | `discovery` | discovery-server dev database |
| `agent_discovery_test` | `discovery` | `discovery` | discovery-server test database |
| `agent_monitor`        | `monitor`   | `monitor`   | monitor-server dev database |
| `agent_monitor_test`   | `monitor`   | `monitor`   | monitor-server test database |
| `keycloak`             | `keycloak`  | `keycloak`  | Keycloak dev database   |

The PostgreSQL superuser is fixed as:

- User: `postgres`
- Password: `devpass`

## Keycloak

The `keycloak` service is used for real-user OIDC integration testing across multiple ACPs projects. It uses the realm definitions under `dev-infra/keycloak/realms/`, and runs a development-mode bootstrap once more after the container starts, converging local ports, the signing algorithm, and the black-box test clients to the current conventions.

### Connection information

| Item | Value |
| ---- | --- |
| Admin console / Realm entry point | `http://localhost:9080` |
| Admin username | `admin` |
| Admin password | `devpass` |
| Database | `keycloak` |
| Dev bootstrap script | [keycloak/bootstrap-dev-keycloak.sh](./keycloak/bootstrap-dev-keycloak.sh) |

### Default imported realms

- `acps-registry`
- `acps-monitor`
- `acps-leader`

The users, roles, and basic client configuration of these realms come from `dev-infra/keycloak/realms/*.json`.  
`dev-infra` additionally takes care of two kinds of "development-mode alignment":

- Ensuring each realm's default signing algorithm is `EdDSA` and that a usable `Ed25519` signing key exists
- Writing back each Web client's `redirectUris` / `webOrigins` according to the local development ports

The currently aligned local Web entry points by default are:

- `registry-web` -> `http://localhost:9001`
- `monitor-web` -> `http://localhost:9009`
- `leader-web` -> `http://localhost:9030`

### CLI and black-box test client conventions

`dev-infra` currently additionally ensures that two **official CLI Device Grant clients** exist:

| Realm | Client ID | Purpose | Key characteristics |
| ---- | ---- | ---- | ---- |
| `acps-registry` | `registry-cli` | `acps-cli auth login` / `acps-cli admin auth login` | public client, Device Authorization Grant enabled, direct grant disabled, targeting `registry-api` |
| `acps-monitor` | `monitor-cli` | `acps-cli monitor auth login` | public client, Device Authorization Grant enabled, direct grant disabled, targeting `monitor-api` |

Notes:

- `registry-cli` and `monitor-cli` are for real-user CLI login; they do not share a session and are not reused across realms
- They do not configure a loopback redirect and do not serve as Authorization Code callback clients
- `monitor-cli` carries the `tenant_id` / `allowed_aics` claims, consistent with `monitor-web`

### Black-box test client conventions

`dev-infra` currently additionally ensures that two **test-only OIDC clients** exist:

| Realm | Client ID | Purpose | Key characteristics |
| ---- | ---- | ---- | ---- |
| `acps-registry` | `registry-e2e` | the `just test e2e` OIDC profile of `registry-server` | public client, direct grant enabled, targeting `registry-api` |
| `acps-monitor` | `monitor-e2e` | the `just test e2e` OIDC profile of `monitor-server` | public client, direct grant enabled, targeting `monitor-api` |
| `acps-leader` | `leader-e2e` | the `just test e2e` OIDC profile of `demo-leader` | public client, direct grant enabled, targeting `leader-api` |

Notes:

- The `*-e2e` here are **OIDC clients**, not user accounts
- They are dedicated to black-box integration testing and do not play the role of a browser login entry point
- The unified naming convention is `<project>-e2e`
- Each project maintains its own e2e client within its own realm, and they are not shared across realms
- When adding new projects later, follow the same rule, for example `leader-e2e`

### Additional conventions for `monitor-e2e`

`monitor-server` not only validates the token's issuer / audience / role, but also relies on the resource scope claims in the token for query filtering. Therefore, in addition to the basic direct grant capability, `monitor-e2e` also gets the business claims aligned with `monitor-web`:

| Claim | Source user attribute | Purpose |
| ---- | ---- | ---- |
| `tenant_id` | Keycloak user attribute `tenant_id` | Tenant-level scope |
| `allowed_aics` | Keycloak user attribute `allowed_aics` | AIC-level scope filtering |

In this way, the OIDC black-box integration testing of `monitor-server` covers not only "whether the signature can be verified", but also:

- whether a viewer can only see its own `allowed_aics`
- whether the operator endpoints only allow operator/admin
- whether cross-realm tokens are rejected

### Default test users

#### `acps-monitor`

| Username | Password | `monitor-api` role | Notes |
| ---- | ---- | ---- | ---- |
| `monitor-viewer` | `demo123` | `viewer` | Preconfigured with `tenant_id=tenant-demo`, `allowed_aics=["AIC-DEMO-001","AIC-DEMO-002"]` |
| `monitor-auditor` | `demo123` | `auditor` | Same as above |
| `monitor-operator` | `demo123` | `operator` | Same as above |
| `monitor-admin` | `demo123` | `admin` | Administrator, does not depend on AIC scope by default |

#### `acps-registry`

| Username | Password | `registry-api` role |
| ---- | ---- | ---- |
| `registry-client` | `demo123` | `CLIENT` |
| `registry-staff` | `demo123` | `STAFF` |
| `registry-admin` | `demo123` | `ADMIN` |

#### `acps-leader`

| Username | Password | `leader-api` role | Notes |
| ---- | ---- | ---- | ---- |
| `leader-user` | `demo123` | `user` | For ordinary real users to submit and read their own session |
| `leader-operator` | `demo123` | `operator` | Can read/cancel other users' sessions and issue elevated stream tokens |
| `leader-admin` | `demo123` | `admin` | Administrator; capabilities cover `operator` |

### Common commands

```bash
# Start Keycloak (idempotently imports realms, and completes CLI client, e2e client, EdDSA, redirectUris)
./dev-infra.sh up keycloak
./dev-infra.sh wait keycloak

# Check Keycloak running status
./dev-infra.sh status keycloak

# View Keycloak logs
./dev-infra.sh logs keycloak --follow
```

### Integration with project test entry points

The test entry points of `registry-server`, `monitor-server`, and `demo-leader` are all already wired to this Keycloak setup:

```bash
cd registry-server
just test e2e -- tests/e2e/test_oidc_keycloak_flow.py

cd ../monitor-server
just test e2e -- tests/e2e/test_oidc_keycloak_flow.py

cd ../demo-leader
just test e2e -- tests/e2e/test_oidc_keycloak_flow.py
```

If a project enables the local OIDC configuration, `just dev bootstrap` / `just test bootstrap` will also automatically bring up `keycloak` and wait for the health check to complete.

## Redis logical database allocation

The shared `dev-redis` exposes only the native port `6379`. Each project is isolated by **logical database number (URL path `/N`)** to prevent development/test data from polluting each other:

| Logical database | Project           | Environment | Example `REDIS_URL`            | Key prefix (in application) |
| ------ | ----------------- | ----------- | ----------------------------- | ------------------ |
| `0`    | `mq-auth-server`  | development | `redis://localhost:6379/0`    | `group_acl:`       |
| `1`    | `mq-auth-server`  | testing     | `redis://localhost:6379/1`    | `group_acl:`       |
| `2`    | `monitor-server`  | development | `redis://localhost:6379/2`    | `amp:`             |
| `3`    | `monitor-server`  | testing     | `redis://localhost:6379/3`    | `amp:`             |

When adding a new project, append a row to this table and reference it in the corresponding `.env.example` / `config/*.toml`; do not reuse an existing logical database number.

## ClickHouse / MinIO naming

- ClickHouse dev database: `amp`; monitor-server test database: `amp_test` (created idempotently by the test suite)
- MinIO dev bucket: `amp-access-archive` (automatically ensured on `dev-infra.sh up minio`)

## Port policy

| Range | Purpose | Example |
| ---- | ---- | ---- |
| **90xx** | ACPs application services (direct host access) | registry `9001`, ca `9003`, discovery `9005`, mq-auth `9007`, monitor `9009`, demo-leader Web `9030` |
| **Native-consistent** | Shared dependencies identical to industry standards | PostgreSQL `5432`, Redis `6379`, RabbitMQ `5671`/`15672`, OpenSearch `9200` |
| **19xxx mapping** | Infra services whose native port falls in 90xx | Kafka `19092`, MinIO `19000`/`19001`, ClickHouse TCP `19010` |

On the first `up` of `kafka` (Redpanda), [dev-infra.sh](./dev-infra.sh) idempotently creates the `amp.*` topics. Host applications connect to `localhost:19092`; containers in the same Docker network use `dev-redpanda:9092`.

## Underlying implementation notes

- `dev-infra.sh` is the recommended entry point
- [compose.yml](./compose.yml) is an underlying implementation detail; it can still be used for troubleshooting and for understanding the orchestration structure
- Day-to-day development documentation and project-level `Justfile` no longer expose `profile` or `dev-*` service names directly

## ClickHouse

The `clickhouse` service provides a columnar database for the `monitor-server` Access feature.

### Connection information

| Item                 | Value          |
| -------------------- | ----------- |
| HTTP port (query)    | `8123`      |
| TCP port (client)    | `19010` (host) → `9000` inside the container |
| Database               | `amp`       |
| Username               | `default`   |
| Password                 | (empty)       |

### How to start

```bash
./dev-infra.sh up clickhouse
./dev-infra.sh wait clickhouse
```

### Test database isolation

`monitor-server` integration tests use a separate database `amp_test` to avoid conflicts with development data. `amp_test` is created automatically by the test suite at startup through `store.ensure_access_schema()` (idempotent via IF NOT EXISTS).

Run the ClickHouse-related integration tests:

```bash
cd monitor-server
./dev-infra.sh up redis clickhouse
./dev-infra.sh wait redis clickhouse
uv run pytest tests/integration/test_access_clickhouse_schema.py \
              tests/integration/test_access_dedupe.py \
              tests/integration/test_access_store_query.py \
              tests/integration/test_access_writer_integration.py \
              -v
```

### Manual connection

```bash
# HTTP API
curl "http://localhost:8123/?query=SELECT+1"

# clickhouse-client (inside the container)
docker exec -it dev-clickhouse clickhouse-client
```

## Kafka (Message / Access)

On the first `up`, the `kafka` (Redpanda) service automatically creates the following topics through [dev-infra.sh](./dev-infra.sh) (idempotent):

| Topic | Partitions | Description |
| --- | --- | --- |
| `amp.access` | 4 | Consumed by the Access Writer (`LogAppendTime`) |
| `amp.message` | 4 | Consumed by the Message Writer (`LogAppendTime`) |
| `amp.message.dlq` | 1 | Message Writer DLQ fallback |

Verification:

```bash
docker exec dev-redpanda rpk topic list | grep amp.message
docker exec dev-redpanda rpk topic describe amp.message
```

For Message module integration testing, see `monitor-server/docs/dev-runbook-message.md`.
