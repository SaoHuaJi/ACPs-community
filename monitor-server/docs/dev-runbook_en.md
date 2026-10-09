**[English](dev-runbook_en.md) | [中文](dev-runbook.md)**

# Development-Mode Joint Verification Runbook: Environment and Service Startup

This document is the shared prerequisite for **development-mode (dev) joint verification** of every log type; it only covers "getting the environment running".

> **Development-mode joint verification**: with each sub-project running under `APP_ENV=development` and sharing
> the local acps-infra infrastructure, walk the whole chain by hand across monitor-server / demo-leader /
> demo-partner / Fluent Bit. Unlike `just test e2e` (pytest automation), this runbook is aimed at
> verification scenarios where **a human operates step by step**.

When verifying a particular log type, first complete service startup per this document, then consult the corresponding dedicated runbook.

## 1. Quick Reference by Log Type

| Log type | Kafka topic | Storage layer | Dedicated document |
|---------|-----------|-------|---------|
| Audit | `amp.audit` | PostgreSQL | [dev-runbook-audit_en.md](./dev-runbook-audit_en.md) (Mock main path; [CA mode](./dev-runbook-audit_en.md#4-advanced-ca-joint-verification-ca-mode) optional) |
| Heartbeat | `amp.heartbeat` | Redis | [dev-runbook-heartbeat_en.md](./dev-runbook-heartbeat_en.md) ([lifecycle verification](./dev-runbook-heartbeat_en.md#4-lifecycle-verification-optional-alive--silent) optional) |
| Metrics | `amp.metrics` | Redis + VictoriaMetrics | [dev-runbook-metrics_en.md](./dev-runbook-metrics_en.md) |
| Access | `amp.access` | ClickHouse | [dev-runbook-access_en.md](./dev-runbook-access_en.md) |
| Message | `amp.message` | ClickHouse | [dev-runbook-message_en.md](./dev-runbook-message_en.md) |
| System | `amp.system` | OpenSearch | [dev-runbook-system_en.md](./dev-runbook-system_en.md) |

> When a new log type is added later, append a row to this table and create `dev-runbook-{type}.md`.

### 1.1 Verification Layers and Script Naming

| Layer | Entry point | Description |
|------|------|------|
| Development-mode runbook | `docs/dev-runbook-*.md` | Walk the whole chain by hand, step by step (this document family) |
| Smoke test | `scripts/smoke_*.py` | Confirm the path within 30s (needs only infra + monitor, no demo dependency); for Access see `smoke_access.py` |
| Full-chain demo | `scripts/demo_*.sh` / `e2e_*_demo.sh` | All services online, wait for natural propagation then assert (see the working-directory note in [§1.2](#12-runtime-caveats)) |
| Automated E2E | `just test e2e` | pytest black-box tests, runnable in CI |

### 1.2 Runtime Caveats

Three operational constraints commonly encountered during joint debugging (recommended to follow by default):

**① Wait for infra to be ready before starting monitor-server**

`just dev start` / `just dev restart` run the same check-first `just dev bootstrap` logic; besides postgres / kafka / redis, they also check whether
the containers of **enabled profiles** such as VictoriaMetrics, ClickHouse and OpenSearch are healthy. On first startup, after
`docker compose up`, or after the machine resumes from sleep, if doctor reports some service as `starting` / `unhealthy`, wait before starting the application:

```bash
cd monitor-server
just infra up postgres kafka redis victoria-metrics clickhouse opensearch
just infra wait postgres kafka redis victoria-metrics clickhouse opensearch
just dev start
```

When verifying only Audit / Heartbeat you may just `wait postgres kafka redis`; to run the full chain (Metrics / Access / Message / System)
you must `up` + `wait` the corresponding storage as well (see the prerequisite checks in each dedicated runbook).

**② Working directory for full-chain scripts**

| Script type | Recommended way to run | Description |
|---------|-------------|------|
| `scripts/smoke_*.py`, `e2e_*_verify.py` | `uv run python ...` after `cd monitor-server` | Does not read demo local log paths |
| `scripts/demo_heartbeat.sh`, `demo_metrics.sh`, `demo_audit.sh` | `bash monitor-server/scripts/...` (any current directory works) | The script resolves the `acps/` repository root itself (`demo-leader` / `demo-partner` log paths) |
| `scripts/e2e_access_demo.sh`, `e2e_message_demo.sh` | any directory works | Only calls HTTP APIs, does not depend on local log paths |
| `scripts/e2e_system_demo.sh` | any directory works | Resolves the absolute path of the partner system log automatically |
| Fluent Bit (§3.5) | **must be in the `acps/` root directory** | The tail paths in the configuration are relative to the repository root |

**③ After discovery-server alive-sync starts, wait a moment before verifying**

After `just dev restart` the HTTP port is available immediately, but alive-sync's bootstrap / Kafka subscription takes about **15–20s**.
Requesting `GET /admin/alive-sync/status` too early may return an empty response or `kafkaNextOffset: null`.
You should wait until the logs show `alive-sync bootstrap completed` or `alive-sync resumed successfully` before running
[dev-runbook-heartbeat-discovery-consumer_en.md](./dev-runbook-heartbeat-discovery-consumer_en.md) §5.

## 2. Prerequisites

The services of each project have been brought up with `just dev start` (or `restart`), and the host machine has installed:

- `uv`, `just`
- Fluent Bit (`brew install fluent-bit`, macOS ≥ v2)
- Docker Desktop (runs the acps-infra infrastructure)
- `redis-cli` (`brew install redis`, used during Heartbeat/Metrics verification to query Redis directly)

> **Redis logical database**: monitor-server in development mode uses `redis://localhost:6379/2` (`config/default.toml`).
> The commands in this runbook that query Redis directly are all written as `redis-cli -n 2`; `redis-cli -p 6379` uses DB 0 by default and **cannot see**
> monitor's watermark and heartbeat data.

Metrics verification also requires starting VictoriaMetrics:

```bash
just infra up victoria-metrics   # or:
docker compose -f acps-infra/dev-infra/compose.yml --profile victoria-metrics up -d
```

Access verification also requires starting ClickHouse:

```bash
just infra up clickhouse   # or:
docker compose -f acps-infra/dev-infra/compose.yml --profile clickhouse up -d
curl -s http://localhost:8123/ping   # expected: Ok.
```

System verification also requires starting OpenSearch:

```bash
just infra up opensearch   # or:
docker compose -f acps-infra/dev-infra/compose.yml --profile opensearch up -d
curl -s 'http://localhost:9200/_cluster/health?pretty'   # expected: status green/yellow (the URL must be quoted to keep zsh from treating ? as a glob)
```

Sibling directory structure:

```text
acps/
├── acps-infra/        # shared infrastructure (Kafka / Redis / PostgreSQL)
├── acps-sdk/          # shared SDK (contains AuditEmitter, HeartbeatEmitter, MetricsEmitter)
├── ca-server/         # certificate and public-key service (needed for Audit CA mode)
├── demo-leader/       # Leader application
├── demo-partner/      # Partner application
└── monitor-server/    # this project
```

## 3. Service Startup

Use a separate terminal for each step and keep it running.

### 3.1 Infrastructure (acps-infra)

`just dev start` / `restart` bring up the infra and run `prep migrate app` (see §3.2). After **wiping Docker volumes / a first clone** you must run
`just dev restart` for monitor-server (see §3.2), to avoid old processes still being connected to an empty database (`/health` may not detect missing tables).

If you only need to check infra status:

```bash
just -f monitor-server/Justfile infra status
# expected: dev-postgres / dev-redpanda / dev-redis all show running + healthy
```

For Kafka heartbeat / Metrics topics you must confirm the timestamp type (mandatory check for the Heartbeat / Metrics runbooks):

```bash
docker exec dev-redpanda rpk topic describe amp.heartbeat -c | grep -i timestamp
# expected: message.timestamp.type  LogAppendTime

docker exec dev-redpanda rpk topic describe amp.metrics -c | grep -i timestamp
# expected: message.timestamp.type  LogAppendTime

docker exec dev-redpanda rpk topic describe amp.access -c | grep -i timestamp
# expected: message.timestamp.type  LogAppendTime

docker exec dev-redpanda rpk topic describe amp.access -p
# expected: PARTITION row count ≥ 4 (if there is only 1 partition, run `just infra up kafka` below)

docker exec dev-redpanda rpk topic describe amp.message -c | grep -E 'timestamp|partition'
# expected: message.timestamp.type  LogAppendTime, partition count ≥ 4
```

If `amp.metrics` is still `CreateTime`, metrics forwarded by Fluent Bit will go to the DLQ (see
[dev-runbook-metrics_en.md §5.1](./dev-runbook-metrics_en.md)). To fix:

```bash
just -f monitor-server/Justfile infra up kafka   # idempotently fixes LogAppendTime
```

Metrics verification requires confirming that VictoriaMetrics is up:

```bash
curl -s http://localhost:8428/health
# expected: Alive (some versions return OK)
```

### 3.2 monitor-server

```bash
cd monitor-server
just dev restart
# restart/start: infra up + wait + prep env/sync/hooks + prep migrate app
# restart: stop the old process and start the API (:9009) + background Writers (Audit/Heartbeat/Metrics/Access/Message/System)
# if doctor fails because infra is not ready, wait first (see §1.2 ①):
# just infra wait postgres kafka redis victoria-metrics clickhouse opensearch
```

When you have merely pulled new migrations in daily work, you can likewise repeat the above command (idempotent).
If you are sure no old process is running, `just dev start` also works.

Verify:

```bash
curl -s http://localhost:9009/health | python3 -m json.tool
# {
#   "status": "ok",
#   "checks": {"database": "ok", "redis": "ok"}
# }
```

During joint verification (including Message / Heartbeat Kafka consumption), it is recommended to set `reload = false` in the `[server]` section of `config/development.toml` and then run `just dev restart`, to keep Uvicorn hot reload from interrupting background tasks. `just dev start` reads that configuration to decide whether to enable `--reload`.

### 3.3 demo-leader

```bash
cd demo-leader
just dev restart
# recommended to run as a pair on first use, after a Docker volume reset, or after pulling config changes
```

Verify:

```bash
curl -s http://localhost:9031/api/v1/health
# {"status":"healthy","version":"1.0.0",...}
```

### 3.4 demo-partner

```bash
cd demo-partner
just dev restart
```

Verify:

```bash
just -f demo-partner/Justfile app status
```

### 3.5 Fluent Bit

A single Fluent Bit process covers every log type (audit + heartbeat + metrics + access + message + system) and uses the same configuration file.
**Run it in the foreground in a dedicated terminal and just keep the window open**:

```bash
# run from the acps/ root directory (the configuration file uses absolute paths)
fluent-bit -c "$(pwd)/monitor-server/config/fluent-bit/fluent-bit.conf"
```

After a successful start you should see the workers of all six Kafka OUTPUTs started:

```
[output:kafka:kafka.0] worker #0 started      # amp.audit
[output:kafka:kafka.1] worker #0 started      # amp.heartbeat
[output:kafka:kafka.2] worker #0 started      # amp.metrics
[output:kafka:kafka.3] worker #0 started      # amp.access
[output:kafka:kafka.4] worker #0 started      # amp.message
[output:kafka:kafka.5] worker #0 started      # amp.system
```

> **macOS caveats**:
> - Every `[OUTPUT]` section must keep `Workers 1`, otherwise the Kafka plugin exits silently due to kqueue compatibility issues.
> - **Do not use `-d` daemon mode**: on macOS `-d` makes the event loop stop working after the fork,
>   so logs cannot actually be forwarded. Just run it in the foreground in a dedicated terminal.
> - **Fluent Bit must be restarted after a configuration change** (for example when a system INPUT/OUTPUT is added): the old process will not load new sections automatically.
>   Stop the old process and run the above command again, then confirm that six workers appear (including `kafka.5` # amp.system).
> - **Access instrumentation depends on acps-sdk `AccessEmitter`**: after `acps-sdk` or demo code is updated you must `just dev restart`
>   demo-leader / demo-partner (see the quick checks in [dev-runbook-access_en.md](./dev-runbook-access_en.md)).

## 4. Common Kafka Check Commands

Topic names and consumer group names for each log type:

| Type | Topic | Consumer group | DLQ topic |
|------|------|--------|---------|
| Audit | `amp.audit` | `amp.audit.writer` | `amp.audit.dlq` |
| Heartbeat | `amp.heartbeat` | `monitor-server.heartbeat.writer.v1` | `amp.heartbeat.dlq` |
| Metrics | `amp.metrics` | `monitor-server.metrics.writer.v1` | `amp.metrics.dlq` |
| Access | `amp.access` | `monitor-server.access.writer.v1` | `amp.access.dlq` |
| Message | `amp.message` | `monitor-server.message.writer.v1` | `amp.message.dlq` |
| System | `amp.system` | `monitor-server.system.writer.v1` | `amp.system.dlq` |

```bash
# view topic partition watermarks (HIGH-WATERMARK increases as logs are written)
docker exec dev-redpanda rpk topic describe {topic} -p

# view consumer group LAG (should approach 0)
docker exec dev-redpanda rpk group describe {consumer-group}

# view DLQ watermarks
# a brand-new environment should be 0; when reusing an existing environment, watch whether the watermark grows during this verification run (no growth means normal)
docker exec dev-redpanda rpk topic describe {topic}.dlq -p
```

## 5. Common Startup Problems

### No logs reaching Kafka after Fluent Bit starts

1. **`-d` daemon mode was used on macOS**: switch to running in the foreground (see §3.5).
2. Confirm that `worker #0 started` appears on standard output.
3. Confirm that every `[OUTPUT]` in the configuration has `Workers 1` set.
4. Confirm that log file paths are absolute and that the files actually exist.
5. Confirm that Redpanda is reachable: `docker exec dev-redpanda rpk cluster health`.

### monitor-server fails to start

```bash
# view live logs
just -f monitor-server/Justfile app logs

# check infrastructure health
just -f monitor-server/Justfile infra status
```

### Infrastructure (Kafka / Redis / PostgreSQL etc.) is not started or not healthy

```bash
cd monitor-server
just infra up postgres kafka redis victoria-metrics clickhouse opensearch
just infra wait postgres kafka redis victoria-metrics clickhouse opensearch
```

If you only need part of the chain, you can shorten the service list of `up` / `wait` (see §1.2 ①). Before `just dev start`,
`redis-cli -n 2 ping` should return `PONG`; OpenSearch may be `starting` right after being brought up, so you must `wait` before starting monitor.

### After a Docker volume reset

After wiping Docker volumes such as Postgres / Redis / Kafka, run **`just dev restart`** for each service in order
(monitor-server, demo-leader, demo-partner; for Message you additionally need `just infra up rabbitmq` and mq-auth-server, see
[dev-runbook-message_en.md](./dev-runbook-message_en.md)). Do not merely kill the processes and restart them without going through `just dev restart` — the development database migrate runs inside the startup preparation logic, and
`just test bootstrap` only migrates the **test database**, so it cannot replace the development-mode `just dev start` / `restart`.
