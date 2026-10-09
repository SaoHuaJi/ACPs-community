**[English](dev-runbook-heartbeat_en.md) | [中文](dev-runbook-heartbeat.md)**

# Development-Mode Heartbeat Joint Verification

**Prerequisites**: first complete service startup (monitor-server, demo-leader, demo-partner, Fluent Bit)
following [dev-runbook_en.md](./dev-runbook_en.md). This document only covers how to verify the Heartbeat pipeline in development mode.

Compared with the Audit pipeline (see [`dev-runbook-audit_en.md`](./dev-runbook-audit_en.md)), Heartbeat has three fundamental differences:

- **Trigger**: heartbeats are background events **emitted periodically as long as the process is online**, and they **require no business API call** (Audit is triggered by business actions).
- **Source of truth**: the current heartbeat state lives in **Redis** (no PostgreSQL, no hash chain, no signature verification).
- **Sync path**: in addition to the Query API, a **Sync API** is provided (full NDJSON snapshot + `alive-delta` Kafka delta stream).

On a first read, we suggest using [the Chapter 4 quick script](#5-quick-verification-scripts) first to confirm the server-side path,
then following [Chapters 2–3](#2-generating-heartbeat-data) to run the whole demo → Fluent Bit → Kafka → Redis → API pipeline end to end.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
demo-leader / demo-partner (one heartbeat every ~15s after the process starts)
  └─ acps-sdk HeartbeatEmitter
        │ writes the NDJSON heartbeat log
        ▼
  logs/amp_heartbeat.jsonl          (demo-leader, one file)
  logs/amp_heartbeat_*.jsonl        (demo-partner, one file per Agent)
        │
        │ Fluent Bit tail + JSON parse
        ▼
  Kafka  amp.heartbeat  (localhost:19092, Redpanda, message.timestamp.type=LogAppendTime)
        │
        │ consumed by monitor-server HeartbeatWriter
        │ observed_at_ms ← Kafka LogAppendTime; hb_apply_heartbeat atomic write
        ▼
  Redis  (liveness source of truth)  localhost:6379
        │   amp:hb:{hb-000}:latest:<aic>      (Hash: latest heartbeat)
        │   amp:hb:{hb-000}:liveness_zset     (ZSet: score=last_seen_at_ms)
        │   amp:hb:{hb-000}:delta_outbox      (Stream: alive-delta pending)
        │
        ├─ Query API ───────────────► http://localhost:9009/acps-amp-v1/heartbeat/liveness|summary
        │
        └─ HeartbeatRelay ──► Kafka amp.heartbeat.alive-delta
                              Sync API ─► http://localhost:9009/acps-amp-v1/heartbeat/sync/info|snapshot
```

Components and ports involved:

| Component | Port | Description |
|------|------|------|
| acps-infra Redpanda (Kafka) | 19092 | Heartbeat event stream (accessed from the host) |
| acps-infra Redis | 6379 | Heartbeat liveness source of truth |
| acps-infra PostgreSQL | 5432 | Needed only for monitor-server startup/health checks (heartbeats themselves never go into PG) |
| demo-leader API | 9031 | Leader business API (health checks; heartbeats are decoupled from business traffic) |
| demo-partner | 9021-9025 | 5 Partner Agent instances (each emits heartbeats periodically) |
| monitor-server Query/Sync API | 9009 | Heartbeat query and sync endpoints |

Kafka topics involved:

| Topic | Partitions | Key configuration | Purpose |
|------|------|---------|------|
| `amp.heartbeat` | **1** | **`message.timestamp.type=LogAppendTime`** | Heartbeat input stream (consumed by the Writer) |
| `amp.heartbeat.alive-delta` | 1 | retention 7d | Delta stream for the alive set (produced by the Relay) |
| `amp.heartbeat.dlq` | 1 | retention 7d | Dead letter for bad messages |

> `amp.heartbeat` must have 1 partition + `LogAppendTime`; both are created idempotently by
> `acps-infra/dev-infra/dev-infra.sh`. If it was ever created or reset manually, be sure to confirm (see §5.1).

### 1.2 Key Differences from the Audit Pipeline

| Dimension | Audit | Heartbeat |
|------|-------|-----------|
| Emitter | `AuditEmitter` (triggered by business instrumentation) | `HeartbeatEmitter` (in-process periodic task) |
| Signed? | Yes (`integrity` always present, signature verified) | No (`integrity` omitted, not verified) |
| Event-time source | The record carries its own `timestamp` | **Kafka LogAppendTime** |
| Storage | PostgreSQL `audit_records` | Redis (liveness Hash + ZSet + outbox Stream) |
| Query endpoints | `/acps-amp-v1/audit/records/query` and others | `/acps-amp-v1/heartbeat/liveness\|summary` + `/heartbeat/sync/*` |
| Dependent services | PostgreSQL + (CA mode) ca-server | Redis |

## 2. Generating Heartbeat Data

### 2.1 Automatic Generation (Default)

After demo-leader / demo-partner start, the background task in each process automatically writes one heartbeat to a local file **every ~15s**,
with no business request needed. Set `AMP_HEARTBEAT_INTERVAL_SECONDS` to override the interval (export it before startup):

```bash
AMP_HEARTBEAT_INTERVAL_SECONDS=10 just -f demo-leader/Justfile app start
```

### 2.2 Inspecting the Local Files Already Written

```bash
tail -1 demo-leader/logs/amp_heartbeat.jsonl | python3 -m json.tool
# {
#   "schema_version": "1.0",
#   "log_type": "heartbeat",
#   "timestamp": "2026-06-12T14:00:00.123456+00:00",
#   "aic": "1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ",
#   "body": {"uptimeSeconds": 45.12}
# }

for f in demo-partner/logs/amp_heartbeat_*.jsonl; do
  echo "== ${f} =="; tail -1 "${f}" | python3 -m json.tool
done
```

### 2.3 Fallback: Emit One Manually (Inline SDK, No demo Needs to Be Running)

```bash
cd demo-leader
uv run python - <<'EOF'
import sys; sys.path.insert(0, '../acps-sdk')
from pathlib import Path
from acps_sdk.amp import HeartbeatEmitter
aic = '1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ'
e = HeartbeatEmitter(Path('logs/amp_heartbeat.jsonl'), aic=aic)
print('emitted log_id:', e.emit_sync())
EOF
```

## 3. Verifying Each Stage

### 3.1 Verify Kafka (Fluent Bit Has Forwarded)

```bash
docker exec dev-redpanda rpk topic describe amp.heartbeat -p
# Expected: HIGH-WATERMARK increases with every heartbeat

# Consume the latest record to confirm its content
HW=$(docker exec dev-redpanda rpk topic describe amp.heartbeat -p | awk '$1=="0"{print $6}')
docker exec dev-redpanda rpk topic consume amp.heartbeat --partitions=0 --offset="$((HW-1))" --num=1 \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(json.loads(d['value']), indent=2, ensure_ascii=False))"
# Expected: log_type == "heartbeat", aic correct

# The DLQ should not grow (if it grows, the topic is not LogAppendTime, see §5.1)
docker exec dev-redpanda rpk topic describe amp.heartbeat.dlq -p
```

### 3.2 Verify monitor-server Has Consumed

```bash
docker exec dev-redpanda rpk group describe monitor-server.heartbeat.writer.v1
# Expected: LAG approaches 0
```

### 3.3 Verify the Current State in Redis

`heartbeat_shard_count=1`, so the shard is always `hb-000`.

```bash
# List heartbeat-related keys
redis-cli -n 2 --scan --pattern 'amp:hb:*' | sort | head

# liveness ZSet: member=aic, score=last_seen_at_ms
redis-cli -n 2 zrange 'amp:hb:{hb-000}:liveness_zset' 0 -1 withscores

# Latest heartbeat Hash for a single AIC
AIC=1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ
redis-cli -n 2 hgetall "amp:hb:{hb-000}:latest:${AIC}"
```

### 3.4 Verify the Query API

```bash
AIC=1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ

# Point query for a single AIC
curl -s "http://localhost:9009/acps-amp-v1/heartbeat/liveness/${AIC}" | python3 -m json.tool
# Expected: data.isAlive=true, data.livenessState="alive", data.silenceDurationSeconds small

# Global summary
curl -s "http://localhost:9009/acps-amp-v1/heartbeat/summary" | python3 -m json.tool
# Expected: data.aliveCount >= number of online Agents; data.silenceBuckets bucketed

# Batch query
curl -s -X POST "http://localhost:9009/acps-amp-v1/heartbeat/liveness/query" \
  -H 'Content-Type: application/json' \
  -d "{\"filter\":{\"conditions\":[{\"field\":\"aic\",\"op\":\"in\",\"value\":[\"${AIC}\"]}]},\"page\":{\"limit\":50}}" \
  | python3 -m json.tool

# Silence ranking (available when analyticsEnabled=true)
curl -s -X POST "http://localhost:9009/acps-amp-v1/heartbeat/silence/top" \
  -H 'Content-Type: application/json' \
  -d '{"topN":10,"onlySilent":false}' | python3 -m json.tool
```

> The response field `meta.dataFreshnessAt` is the event-time watermark that the read model has processed up to; `meta.evaluatedAt`
> is the time of this evaluation, and `meta.silenceThresholdSeconds`/`evictAfterSeconds` echo the thresholds.

### 3.5 Verify the Sync API

```bash
# Sync Profile metadata
curl -s "http://localhost:9009/acps-amp-v1/heartbeat/sync/info" | python3 -m json.tool
# Expected: type="amp-alive-delta", kafkaTopic="amp.heartbeat.alive-delta",
#           shardCount=1, currentPublishedSeqByShard={"hb-000":"<n>"}

# Full snapshot (NDJSON: first line snapshot-meta, then upsert lines)
curl -s "http://localhost:9009/acps-amp-v1/heartbeat/sync/snapshot"
# First line {"recordType":"snapshot-meta","type":"amp-alive-delta","cutoverSeqByShard":{...},...}
# Later lines {"shard":"hb-000","seq":"..","op":"upsert","id":"urn:amp:alive:<aic>",...}

# alive-delta stream: consume the envelopes produced by the Relay
docker exec dev-redpanda rpk topic consume amp.heartbeat.alive-delta --num 5 --offset start --format '%v\n' \
  | python3 -c "import sys,json
for line in sys.stdin:
    line=line.strip()
    if line:
        v=json.loads(line); print(v['kind'], v['op'], v['id'])"
# Expected: enter_alive / refresh_alive upsert urn:amp:alive:<aic>
```

> Once the Sync API is confirmed working, you can move on to integrating discovery-server as a Consumer:
> see [`dev-runbook-heartbeat-discovery-consumer_en.md`](./dev-runbook-heartbeat-discovery-consumer_en.md).

## 4. Lifecycle Verification (Optional: alive → silent)

Stop one Agent, wait longer than the silence threshold (`silence_threshold_seconds=90`), and confirm that it is judged silent.

```bash
AIC=1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ
just -f demo-leader/Justfile app stop      # stop the leader; heartbeats stop
sleep 95                                    # wait > 90s silence threshold + Reconciler scan

curl -s "http://localhost:9009/acps-amp-v1/heartbeat/liveness/${AIC}" | python3 -m json.tool
# Expected: data.isAlive=false, data.livenessState="silent", silenceDurationSeconds >= 90
```

If you want to confirm this on the delta stream, the corresponding `alive-delta` appends an envelope with
`kind="leave_alive"` and `op="delete"`—use the consume command from §3.5 to view it from the start, and
the end of the sequence is that AIC's `leave_alive`.

Recovery: `just -f demo-leader/Justfile app start`; after waiting one heartbeat cycle, querying again should return to `alive`.

## 5. Quick Verification Scripts

### 5.1 Quick Path Verification (Skipping Fluent Bit / demo)

`scripts/smoke_heartbeat.py` connects directly to Kafka to deliver one heartbeat and polls the Query API to confirm alive;
it needs only **infra (kafka+redis) + monitor-server** online, verifying the "message→Redis→API" path in the shortest way.

```bash
cd monitor-server
APP_ENV=development uv run python scripts/smoke_heartbeat.py
# [OK] heartbeat delivered: aic=urn:test:heartbeat:e2e, partition=0
# [OK] liveness: isAlive=True, livenessState=alive
# [PASS] Heartbeat message→Redis→Query API path verification passed!
```

### 5.2 One-Click Full-Pipeline Script

`scripts/demo_heartbeat.sh` assumes all services are started and demo is emitting heartbeats periodically;
it waits for the heartbeats to propagate naturally and then asserts liveness.

```bash
# Run from the acps/ root directory (recommended); the script resolves the repository root path automatically, so any current directory works
bash monitor-server/scripts/demo_heartbeat.sh
# Expected to end with ✓ PASS, aliveCount >= 1 and leader isAlive=true
```

## 6. Troubleshooting

### 6.1 All Heartbeats Go to the DLQ (Most Common)

**Symptom**: the `amp.heartbeat.dlq` watermark keeps growing and the Query API cannot find alive; the monitor log shows
`message missing timestamp, skipping retries and writing directly to the DLQ` (`UntimedHeartbeatError`).

**Root cause**: `amp.heartbeat` is not `LogAppendTime`. The heartbeat wire body does **not** contain `observedTimestamp`;
the Writer's precedence for deriving observed_at is LogAppendTime → observedTimestamp → DLQ. If the topic uses the default
`CreateTime`, then the consumer side sees `timestamp_type=0`, both paths fail → DLQ.

**Fix**:

```bash
docker exec dev-redpanda rpk topic describe amp.heartbeat -c | grep -i timestamp
docker exec dev-redpanda rpk topic alter-config amp.heartbeat --set message.timestamp.type=LogAppendTime
# Or have dev-infra rebuild it per the spec:
just -f monitor-server/Justfile infra reset kafka
```

### 6.2 Writer Refuses to Start: Partition Count Mismatch (C-CONF-1)

**Symptom**: monitor fails at startup with `heartbeat_input_partition_count=1 does not match the actual partition count N of topic 'amp.heartbeat'`.

**Fix**: delete it and recreate it with 1 partition:

```bash
docker exec dev-redpanda rpk topic delete amp.heartbeat
just -f monitor-server/Justfile infra reset kafka
```

### 6.3 liveness Stays silent / Cannot Be Found

1. Confirm the local file is growing (§2.2); if it is not growing, the demo heartbeat task did not start:
   ```bash
   just -f demo-leader/Justfile app logs | grep -i heartbeat
   ```
2. Confirm the emit interval < 90s (default 15s).
3. Confirm Fluent Bit is forwarding (§3.1) and the DLQ is not growing (§6.1).
4. Confirm the Writer is consuming (§3.2, consumer group lag → 0).

### 6.4 Redis Not Started / Query Returns 503

```bash
redis-cli -n 2 ping                           # Expected PONG
just -f monitor-server/Justfile infra up redis
just -f monitor-server/Justfile infra status      # redis healthy
```

The Query API returns `503` by design when the Redis connection is abnormal, and works normally again once Redis recovers.

### 6.5 No Heartbeats Reach Kafka After Fluent Bit Starts

See the general troubleshooting in [dev-runbook_en.md §5](./dev-runbook_en.md), plus the Heartbeat-specific items:

1. **`-d` daemon mode was used on macOS**: switch to running in the foreground (see `dev-runbook_en.md §3.5`).
2. Check the heartbeat OUTPUT worker: the standard output should show `[output:kafka:kafka.1] worker #0 started`.
3. Confirm the configuration contains the heartbeat section (INPUT tag=`amp.heartbeat` + OUTPUT match=`amp.heartbeat` + `Workers 1`).
4. Confirm the heartbeat file path is correct (absolute path): `ls demo-leader/logs/amp_heartbeat.jsonl`.

### 6.6 `/sync/*` Returns 404

When `sync_enabled=false`, `/sync/info` and `/sync/snapshot` return 404 (expected behavior).
Development defaults to `sync_enabled=true`; if it was turned off, set `sync_enabled = true` under `[heartbeat]`
in `config/default.toml` and then restart monitor-server.

### 6.7 `/silence/top` Returns 404

When `analytics_enabled=false` this endpoint is **not registered** (the 404 means the route does not exist, not a business error).
Development defaults to `analytics_enabled=true`.

## 7. Relationship to Automated Tests

| Layer | Entry point | Description |
|------|------|------|
| Manual joint verification | This document | Multiple projects + Fluent Bit + real demo periodic heartbeats |
| Quick path script | `scripts/smoke_heartbeat.py` | Direct Kafka connection, skipping Fluent Bit/demo |
| Full-pipeline one-click | `scripts/demo_heartbeat.sh` | All services online, asserts liveness |
| pytest e2e | `just test e2e` (`tests/e2e/test_heartbeat_*`) | Black box: deliver→liveness/summary/query/sync, requires `just test bootstrap` |

> For code locations and implementation design, see [`plans/heartbeat-e2e-demo-plan.md`](../plans/heartbeat-e2e-demo-plan.md).

> **discovery-server integration**: if you need to verify the complete path of discovery-server as a Heartbeat Sync API Consumer
> (bootstrap → aliveMap injection), see
> [`dev-runbook-heartbeat-discovery-consumer_en.md`](./dev-runbook-heartbeat-discovery-consumer_en.md).
