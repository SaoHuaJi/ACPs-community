**[English](dev-runbook-heartbeat-discovery-consumer_en.md) | [中文](dev-runbook-heartbeat-discovery-consumer.md)**

# Development-Mode Joint Verification: discovery-server as a Heartbeat Sync API Consumer

**Prerequisite**: first follow [dev-runbook-heartbeat_en.md](./dev-runbook-heartbeat_en.md) to get the
monitor-server Heartbeat chain working end to end (at least complete the §3.4 Query API + §3.5 Sync API
verification, confirming that `/acps-amp-v1/heartbeat/sync/info` returns normally and that the
`amp.heartbeat.alive-delta` topic has data).
On that basis, this document independently describes the joint debugging steps for **connecting
discovery-server to the Sync API as a Consumer**.

Compared with the Heartbeat Runbook, this document adds the following chain segments:

- **Provider → Consumer**: monitor-server exposes the full snapshot externally through `/sync/info`
  and `/sync/snapshot`, and continuously publishes delta envelopes through the
  `amp.heartbeat.alive-delta` topic.
- **Consumer local persistence**: discovery-server consumes the snapshot and deltas, atomically
  persists them to PostgreSQL (`agent_alive_status` + `alive_sync_shard_state`), and attaches
  `aliveMap` to the `result` field in every `POST /acps-adp-v2/discover` response.

---

## 1. Chain Overview

### 1.1 Data Flow and Ports

```text
demo-leader / demo-partner (periodic process heartbeats)
  └─ acps-sdk HeartbeatEmitter → logs/amp_heartbeat*.jsonl
        │ Fluent Bit → Kafka amp.heartbeat
        ▼
  monitor-server HeartbeatWriter → Redis liveness
        │
        ├─ HeartbeatRelay ──► Kafka  amp.heartbeat.alive-delta
        │                      (enter_alive / refresh_alive / leave_alive envelopes)
        │
        └─ Sync API (Provider)
               GET /acps-amp-v1/heartbeat/sync/info       ← metadata (topic, shard)
               GET /acps-amp-v1/heartbeat/sync/snapshot   ← full NDJSON snapshot
               │
               │ discovery-server heartbeat_sync (Consumer)
               │   Phase 1 bootstrap: AliveSyncSourceClient
               │     → /sync/info verifies the Provider is online
               │     → /sync/snapshot streamed pull, apply_snapshot writes to PG
               │   Phase 2 delta consumption: AliveDeltaKafkaConsumer
               │     → subscribe to amp.heartbeat.alive-delta
               │     → engine.poll_apply: upsert / delete alive rows
               │   Phase 3 resync: seq gap or 503 detected → reset → backoff → bootstrap
               ▼
  PostgreSQL  agent_alive_status (AIC liveness table)
              alive_sync_shard_state (per-shard checkpoint)
               │
               └─ POST /acps-adp-v2/discover
                      response result.aliveMap = {aic: {alive, lastSeenAt}}
```

Components and ports involved:

| Component | Port | Description |
|------|------|------|
| acps-infra Redpanda (Kafka) | 19092 | alive-delta topic |
| acps-infra PostgreSQL | 5432 | discovery DB (alive state persistence) |
| monitor-server Sync/Query API | 9009 | Provider: `/sync/info` + `/sync/snapshot` |
| discovery-server | 9005 | Consumer; discover + admin alive-sync API |

Kafka topics involved:

| Topic | Description |
|------|------|
| `amp.heartbeat.alive-delta` | Produced by monitor-server HeartbeatRelay; subscribed to by discovery-server |

### 1.2 Three-Phase Bootstrap Mechanism

When starting alive-sync, discovery-server bootstraps itself in the following order:

| Phase | Description |
|------|------|
| **① bootstrap** | GET `/sync/info` (verify the Provider is online) → GET `/sync/snapshot` (NDJSON streamed read, atomic write to PG) → compute the Kafka seek plan (cutover + 5 min lookback margin) |
| **② delta consumption** | Kafka `poll_apply`: `enter_alive / refresh_alive` → upsert; `leave_alive` → `alive=false`; each record advances the checkpoint in the same transaction |
| **③ resync** | Seq gap or 503 degradation detected → `store.reset()` (clear both tables) → back off 10s → redo ① |

When a checkpoint already exists, **resume** takes priority (skip the snapshot and continue directly from
the Kafka offset), falling back to a full bootstrap only when hydrate fails.

---

## 2. Prerequisites

1. The [dev-runbook-heartbeat_en.md §3.5](./dev-runbook-heartbeat_en.md) Sync API verification is complete:

   ```bash
   curl -s "http://localhost:9009/acps-amp-v1/heartbeat/sync/info" | python3 -m json.tool
   # Expected: type="amp-alive-delta", kafkaTopic="amp.heartbeat.alive-delta", shardCount=1
   ```

2. The `amp.heartbeat.alive-delta` topic has data (monitor-server Relay has produced at least one envelope):

   ```bash
   docker exec dev-redpanda rpk topic describe amp.heartbeat.alive-delta -p
   # Expected: HIGH-WATERMARK >= 1
   ```

3. discovery-server has already run `just dev start` (DB migrated, including the two alive-sync tables).

   > If bootstrap fails at `CREATE EXTENSION vector` (`permission denied`), create the extension
   > in the `agent_discovery` database as the postgres superuser beforehand and retry:
   >
   > ```bash
   > psql postgresql://postgres:devpass@localhost:5432/agent_discovery \
   >   -c "CREATE EXTENSION IF NOT EXISTS vector;"
   > cd discovery-server && just dev start
   > ```

4. **Development-mode environment variables**: `APP_ENV=development` in `.env` (otherwise `[alive_sync]`
   in `config/development.toml` is not loaded). Confirm with the following command:

   ```bash
   cd discovery-server && APP_ENV=development uv run python - <<'EOF'
   from app.core.config import settings
   print('APP_ENV:', settings.APP_ENV)
   print('ALIVE_SYNC_ENABLED:', settings.ALIVE_SYNC_ENABLED)
   EOF
   # Expected: APP_ENV: development, ALIVE_SYNC_ENABLED: True
   ```

5. **Heartbeat data is flowing**: there should currently be online Agents on the monitor-server side
   (`aliveCount > 0`). If `curl -s http://localhost:9009/acps-amp-v1/heartbeat/summary` shows
   `aliveCount: 0`, first troubleshoot according to
   [dev-runbook-heartbeat_en.md](./dev-runbook-heartbeat_en.md), and confirm that Fluent Bit is
   running in the foreground as described in [dev-runbook_en.md §3.5](./dev-runbook_en.md).

6. **Discover sample data** (needed for the §5.4 aliveMap verification): the discovery DB must contain
   Agent/Skill indexes, otherwise the discover response contains no AIC and `aliveMap` cannot be injected:

   ```bash
   cd discovery-server && just prep seed app
   ```

---

## 3. Configuring alive-sync

Append the following at the end of `discovery-server/config/development.toml` (if the `[alive_sync]`
section does not yet exist):

```toml
[alive_sync]
enabled                    = true
auto_start                 = true
provider_base_url          = "http://localhost:9009/acps-amp-v1/heartbeat"
kafka_bootstrap_servers    = "localhost:19092"
# leave kafka_topic empty: it is fetched automatically from the kafkaTopic field of /sync/info at startup (recommended)
# the remaining parameters keep their defaults (adjust as needed)
# bootstrap_lookback_seconds     = 300    # Kafka seek lookback margin (seconds), default 5 minutes
# resync_backoff_seconds         = 10     # resync backoff interval (seconds)
# retry_interval_seconds         = 30     # 503 degradation retry interval (seconds)
```

Also, in the `[server]` section of `config/development.toml` it is recommended to set `reload = false`.
The alive-sync background Kafka consumption task is frequently interrupted under Uvicorn hot-reload
mode; turning reload off during joint verification keeps consumption stable.

> **Note**: `provider_base_url` must **not** end with `/`, and the path is `heartbeat`
> (without `/sync/...`). discovery-server appends `/sync/info` and `/sync/snapshot` to it.

It can also be overridden through environment variables (in `.env` or exported at startup),
with higher priority than TOML:

```bash
export ALIVE_SYNC_ENABLED=true
export ALIVE_SYNC_PROVIDER_BASE_URL=http://localhost:9009/acps-amp-v1/heartbeat
export ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS=localhost:19092
```

---

## 4. Starting discovery-server and Confirming alive-sync Is Up

```bash
cd discovery-server
just dev restart   # restart if already running (loads the new config)
# first start:
# just dev start
```

Check the startup logs to confirm the three-phase bootstrap completed:

```bash
just -f discovery-server/Justfile app logs
# Expected (in order):
# "alive-sync background task started successfully"
# "alive-sync bootstrap started"
# "alive-sync bootstrap completed, cutover=..."
```

> `alive-sync background task started successfully` means all guard conditions are satisfied
> (`ALIVE_SYNC_ENABLED=true`, `ALIVE_SYNC_AUTO_START=true`, `ALIVE_SYNC_PROVIDER_BASE_URL` configured,
> and not in testing mode). If the logs only show the word "skipped", check whether the §3 configuration
> has been saved and the restart took effect.

**Wait after startup**: after the process starts listening on `:9005`, the alive-sync snapshot pull and
Kafka subscription still take about **15–20s** (see
[dev-runbook_en.md §1.2 ③](./dev-runbook_en.md#12-runtime-caveats)). During this period
`curl .../admin/alive-sync/status` may return an empty body or `kafkaNextOffset: null`; this is normal.
Proceed to the §5 verification only after the logs show `alive-sync bootstrap completed` or
`alive-sync resume succeeded`.

---

## 5. Verifying Each Stage

### 5.1 Admin API: Service Status

```bash
curl -s http://localhost:9005/admin/alive-sync/status | python3 -m json.tool
# Expected:
# {
#   "running": true,
#   "aliveCount": <N>,          ← number of rows with alive=true in PG (> 0 after bootstrap completes)
#   "checkpointCount": 1,       ← number of shards (1 by default in development)
#   "shards": {
#     "hb-000": {
#       "lastSeenSeq": <N>,
#       "cutoverSeq": <N>,
#       "kafkaNextOffset": <N>,
#       "snapshotGeneratedAt": "2026-..."
#     }
#   }
# }
```

`aliveCount > 0` and `kafkaNextOffset` having a value means bootstrap succeeded and delta consumption has started.

### 5.2 PostgreSQL: Local alive State

```bash
# alive state table (written by bootstrap)
psql postgresql://discovery:discovery@localhost:5432/agent_discovery \
  -c "SELECT aic, alive, last_seen_at, shard FROM agent_alive_status
      ORDER BY last_seen_at DESC LIMIT 10;"
# Expected: the demo AIC can be found, alive=true, last_seen_at close to the current time

# checkpoint table (per-shard progress)
psql postgresql://discovery:discovery@localhost:5432/agent_discovery \
  -c "SELECT shard, last_seen_seq, cutover_seq, kafka_next_offset
      FROM alive_sync_shard_state;"
# Expected: shard=hb-000, last_seen_seq / kafka_next_offset increase with heartbeats
```

### 5.3 Kafka Consumer Group

```bash
docker exec dev-redpanda rpk group describe discovery-server.alive-sync.v1
# Expected: STATE=Stable, MEMBERS=1, LAG approaching 0 (delta consumption is timely)
```

> If `STATE=Dead`, alive-sync may be in a resync backoff or bootstrap retry gap (see §8.6);
> re-check after waiting 30s. Keep `reload = false` during joint verification (§3).

### 5.4 Discover Interface: aliveMap Injection

For any discover request, `result.aliveMap` in the response should carry the liveness state of each AIC
found by this request:

```bash
curl -s -X POST http://localhost:9005/acps-adp-v2/discover \
  -H 'Content-Type: application/json' \
  -d '{"query": "Book a high-speed train from Beijing to Shanghai"}' \
  | python3 -m json.tool
# Expected in the result field:
# "aliveMap": {
#   "1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ": {
#     "alive": true,
#     "aliveLastSeenAt": "2026-..."
#   },
#   ...
# }
```

> `aliveMap` contains only the **AICs hit by this query**; AICs not hit do not appear.
> If the returned result is a forwarded response (not produced locally), `aliveMap` is passed through
> unchanged and is not overwritten by discovery-server.
> If alive-sync is not enabled, the `aliveMap` field is absent (zero-breakage compatibility).

### 5.5 Confirming Incremental Deltas Are Being Consumed

Wait for one heartbeat cycle (about 15s), then query the checkpoint again to confirm that `last_seen_seq`
and `kafka_next_offset` have advanced:

```bash
psql postgresql://discovery:discovery@localhost:5432/agent_discovery \
  -c "SELECT shard, last_seen_seq, kafka_next_offset FROM alive_sync_shard_state;"
# Compare two consecutive runs; the values should increase
```

---

## 6. Lifecycle Verification (Optional: alive → silent → alive)

Stop one Agent and verify that the `alive` value of the corresponding AIC in `aliveMap` becomes `false`:

```bash
AIC=1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ
just -f demo-leader/Justfile app stop      # stop leader, heartbeats stop being sent

# Wait for monitor-server to judge it silent and emit a leave_alive delta (> silence_threshold=90s)
sleep 95

# Confirm that monitor-server has emitted leave_alive (consume the latest messages, to avoid --offset start scanning everything and hanging)
HW=$(docker exec dev-redpanda rpk topic describe amp.heartbeat.alive-delta -p | awk '$1=="0"{print $6}')
docker exec dev-redpanda rpk topic consume amp.heartbeat.alive-delta \
  --partitions=0 --offset="$((HW-50))" --num=50 --format '%v\n' | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if line:
        v = json.loads(line)
        if v.get('id','').endswith('${AIC}'):
            print(v['kind'], v['op'], v['id'])"
# Expected at the end: leave_alive delete urn:amp:alive:<AIC>

# If the discovery-server side has not updated in time, trigger a manual resync (§7.1) and then query PG / discover again

# Confirm that the discovery-server side has applied it
psql postgresql://discovery:discovery@localhost:5432/agent_discovery \
  -c "SELECT aic, alive, last_seen_at FROM agent_alive_status WHERE aic='${AIC}';"
# Expected: alive=false (the row is kept after leave_alive), or the row is removed from the snapshot after resync (0 rows)

# Confirm that aliveMap in the discover response has been updated
curl -s -X POST http://localhost:9005/acps-adp-v2/discover \
  -H 'Content-Type: application/json' \
  -d '{"query": "Book a high-speed train from Beijing to Shanghai"}' \
  | python3 -c "
import json, sys
resp = json.load(sys.stdin)
alive_map = (resp.get('result') or {}).get('aliveMap') or {}
status = alive_map.get('${AIC}', '(not in result)')
print('aliveMap[${AIC}]:', status)"
# Expected: alive=false or this AIC is not in the current result
```

Recovery: restart demo-leader, wait one heartbeat cycle and discover again; `alive` should return to `true`:

```bash
just -f demo-leader/Justfile app start
sleep 20   # wait one heartbeat cycle + delta propagation
curl -s -X POST http://localhost:9005/acps-adp-v2/discover \
  -H 'Content-Type: application/json' \
  -d '{"query": "Book a high-speed train from Beijing to Shanghai"}' \
  | python3 -c "
import json, sys
resp = json.load(sys.stdin)
alive_map = (resp.get('result') or {}).get('aliveMap') or {}
print('aliveMap:', json.dumps(alive_map, ensure_ascii=False, indent=2))"
```

---

## 7. Manual Management Operations

### 7.1 Triggering a Manual Resync

Useful when local state must be rebuilt immediately after the Provider recovers, or when debugging the
bootstrap flow:

```bash
curl -s -X POST http://localhost:9005/admin/alive-sync/resync | python3 -m json.tool
# Expected: {"message": "resync triggered"}
```

After triggering, the discovery-server logs should show:

```
alive-sync resync requested, reason: admin_manual_trigger
alive-sync resync backing off for 10 seconds
alive-sync bootstrap started
alive-sync bootstrap completed, cutover=...
```

### 7.2 Temporarily Disabling It (Without Stopping the Service)

```bash
# Set enabled = false in config/development.toml, then restart
# or through an environment variable:
ALIVE_SYNC_ENABLED=false just -f discovery-server/Justfile app restart
```

After disabling, the `aliveMap` field is absent from `discover` responses, and all other behavior is unaffected.

---

## 8. Troubleshooting

### 8.1 alive-sync Not Started (Logs Only Show "skipped")

**Symptom**: `/admin/alive-sync/status` returns `{"running": false, ...}`, and "skipped" appears in the logs.

**Troubleshooting order**:

```bash
# Confirm the configuration took effect
cd discovery-server && APP_ENV=development uv run python - <<'EOF'
from app.core.config import settings
print('ALIVE_SYNC_ENABLED:', settings.ALIVE_SYNC_ENABLED)
print('ALIVE_SYNC_AUTO_START:', settings.ALIVE_SYNC_AUTO_START)
print('ALIVE_SYNC_PROVIDER_BASE_URL:', settings.ALIVE_SYNC_PROVIDER_BASE_URL)
print('ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS:', settings.ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS)
print('APP_ENV:', settings.APP_ENV)
EOF
```

Check each guard condition:

| Condition | Expected Value | Fix |
|------|--------|---------|
| `ALIVE_SYNC_ENABLED` | `True` | set `enabled = true` in `config/development.toml` |
| `ALIVE_SYNC_AUTO_START` | `True` | set `auto_start = true` in `config/development.toml` |
| `ALIVE_SYNC_PROVIDER_BASE_URL` | non-empty http(s) address | add `provider_base_url` |
| `APP_ENV` | `development` | set `APP_ENV=development` in `.env` and restart |
| `UVICORN_RELOAD` | `False` (recommended) | set `reload = false` in `config/development.toml` and restart |

### 8.2 bootstrap Failure: Provider Unreachable (PROVIDER_UNAVAILABLE / CONNECTION_FAIL)

**Symptom**: `AliveSyncError(PROVIDER_UNAVAILABLE)` or `CONNECTION_FAIL` appears in the logs;
`/admin/alive-sync/status` returns `running: false`.

**Troubleshooting**:

```bash
# Confirm monitor-server is online
curl -s http://localhost:9009/health | python3 -m json.tool

# Confirm the Sync API is enabled (sync_enabled=true is the monitor-server development default)
curl -s http://localhost:9009/acps-amp-v1/heartbeat/sync/info | python3 -m json.tool
# If it returns 404: monitor-server has sync_enabled=false, see dev-runbook-heartbeat_en.md §6.6

# Confirm provider_base_url is configured correctly (it must not end with /sync/... or a redundant /)
# Correct: http://localhost:9009/acps-amp-v1/heartbeat
# Wrong: http://localhost:9009/acps-amp-v1/heartbeat/
```

### 8.3 bootstrap Failure: snapshot Unavailable (SNAPSHOT_UNAVAILABLE / DELTA_LOG_UNHEALTHY)

**Symptom**: `/sync/snapshot` returns 503, and the logs contain `SNAPSHOT_UNAVAILABLE` or `DELTA_LOG_UNHEALTHY`.

**Meaning**: the monitor-server Relay or snapshot materialization is not ready yet (usually a brief
occurrence right after startup). discovery-server retries automatically according to
`retry_interval_seconds` (30s by default), so **no manual intervention is needed**.

If it does not recover for a long time, troubleshoot monitor-server:

```bash
# Confirm whether HeartbeatRelay is running
just -f monitor-server/Justfile app logs | grep -i relay

# Confirm the alive-delta topic has data
docker exec dev-redpanda rpk topic describe amp.heartbeat.alive-delta -p
```

### 8.4 Kafka Consumer Group Not Advancing (LAG Keeps Growing)

**Symptom**: `docker exec dev-redpanda rpk group describe discovery-server.alive-sync.v1`
shows LAG growing continuously instead of approaching 0.

**Troubleshooting**:

```bash
# Confirm the Kafka bootstrap servers are configured correctly
cd discovery-server && APP_ENV=development uv run python -c "
from app.core.config import settings
print('ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS:', settings.ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS)"
# Expected in development: localhost:19092

# Confirm Redpanda is reachable
docker exec dev-redpanda rpk cluster health

# Check the discovery-server logs for Kafka errors
just -f discovery-server/Justfile app logs | grep -i kafka
```

If the consumer group was never created (`rpk group describe` reports an error), the Kafka consumer
failed already at the connection stage. Check whether `ALIVE_SYNC_KAFKA_BOOTSTRAP_SERVERS` points to a
port reachable from the host (`localhost:19092` rather than the in-container address `dev-redpanda:9092`).

### 8.5 aliveMap Is Always Empty or Missing

Troubleshoot in order:

1. **alive-sync not enabled**: check whether `running` in `/admin/alive-sync/status` is `true`.
2. **bootstrap not finished yet**: `aliveCount=0` means there are no alive rows in PG; wait or trigger a resync (§7.1).
3. **The AIC hit by the query does not exist in PG**:

   ```bash
   psql postgresql://discovery:discovery@localhost:5432/agent_discovery \
     -c "SELECT COUNT(*) FROM agent_alive_status WHERE alive=true;"
   # If 0, the bootstrap result is empty or the deltas have not caught up
   ```

4. **Discover result comes from forwarding (not produced locally)**: forwarded responses carry the
   upstream `aliveMap` and are not overwritten by discovery-server (ADP §4.2.3 pass-through semantics,
   which is normal behavior).

### 8.6 resync Loops Endlessly (Logs Keep Showing "resync backoff")

**Symptom**: `alive-sync ResyncRequired` or `alive-sync resync requested` appears frequently in the logs.

**Common causes**:

1. **seq gap**: the Kafka `amp.heartbeat.alive-delta` topic was truncated (`LogStartOffset` jumped), and
   discovery-server detects that the seq is discontinuous → triggers resync.
   Check the topic watermarks:

   ```bash
   docker exec dev-redpanda rpk topic describe amp.heartbeat.alive-delta -p
   # Whether LOGSTART-OFFSET is greater than the last consumed offset
   ```

2. **monitor-server restarted + snapshot content changed**: the cutover of the first bootstrap after
   self-bootstrapping does not match the checkpoint of the resume. In this case, simply let the resync
   finish naturally.

3. **Clock skew**: `bootstrap_lookback_seconds` is too small, so messages before the cutover are missed
   during the seek. Increase it (for example to 600s): `config/development.toml` → `bootstrap_lookback_seconds = 600`.

---

## 9. Relationship to Automated Tests

| Layer | Entry Point | Description |
|------|------|------|
| Manual joint verification | This document | Full monitor-server + discovery-server chain, operated step by step by a human |
| discovery-server unit tests | `just -f discovery-server/Justfile test unit` | heartbeat_sync module fully mocked, no external dependencies |
| discovery-server integration tests | `just -f discovery-server/Justfile test integration` | Requires PostgreSQL; alive-sync auto-start is skipped with `APP_ENV=testing` and injected via fixtures |
| discovery-server e2e tests | `just -f discovery-server/Justfile test e2e alive_sync` | Requires PostgreSQL; the alive-sync flow simulates the Provider and Relay through mock HTTP/Kafka |

> Development-mode joint verification and automated tests are complementary: automated tests replace the
> real Provider and Kafka with mocks/stubs in CI, whereas this document covers the **real multi-service
> collaboration scenario** (monitor-server Relay → Kafka → discovery-server bootstrap → aliveMap
> injection), ensuring cross-service protocol contract and configuration correctness.
