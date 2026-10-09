**[English](dev-runbook-metrics_en.md) | [中文](dev-runbook-metrics.md)**

# Development-Mode Metrics Joint Verification

**Prerequisite**: First follow [dev-runbook_en.md](./dev-runbook_en.md) to complete service startup (monitor-server, demo-leader,
demo-partner, Fluent Bit, VictoriaMetrics). This document covers only how to verify the Metrics pipeline in development mode.

**Quick check before starting verification** (if any item is not satisfied, fix it before continuing):

```bash
# 1. amp.metrics must be LogAppendTime
docker exec dev-redpanda rpk topic describe amp.metrics -c | grep timestamp.type
# Expect: LogAppendTime (otherwise run just -f monitor-server/Justfile infra up kafka)

# 2. Fluent Bit must include the kafka.2 worker (metrics OUTPUT)
# After start/restart, stdout should show: [output:kafka:kafka.2] worker #0 started

# 3. demo processes must include the metrics background task (partner must be restarted after a code update)
just -f demo-leader/Justfile app logs | grep "AMP metrics started"
just -f demo-partner/Justfile app logs | grep "AMP metrics started"
```

Compared with the Heartbeat pipeline (see [`dev-runbook-heartbeat_en.md`](./dev-runbook-heartbeat_en.md)), Metrics has three fundamental differences:

- **Dual-write storage**: sampled metrics are written to the **Redis snapshot cache** (latest state) and **VictoriaMetrics** (time-series source of truth). Heartbeat writes only to Redis.
- **Batched Remote Write**: MetricsWriter accumulates a batch (5s or 10k samples) and then performs a single Remote Write, rather than writing record by record.
- **Five endpoints, no Sync**: the Query API provides 5 endpoints (snapshots/series/rankings/slo/capacity) and **no** Sync plane like Heartbeat's.

For a first read, we suggest using the [Chapter 4 quick scripts](#4-quick-verification-scripts) to confirm the end-to-end path,
then following [Chapters 2–3](#2-generating-metrics-data) to run the full chain: demo → Fluent Bit → Kafka → Writer → Redis + VM → API.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
demo-leader / demo-partner (one metrics log every ~30s after the process starts)
  └─ acps-sdk MetricsEmitter
        │ write NDJSON metrics log (includes the resource field)
        ▼
  logs/amp_metrics.jsonl           (demo-leader, one file)
  logs/amp_metrics_*.jsonl         (demo-partner, one file per Agent)
        │
        │ Fluent Bit tail + JSON parse (third Kafka OUTPUT kafka.2)
        ▼
  Kafka  amp.metrics  (localhost:19092, Redpanda, message.timestamp.type=LogAppendTime)
        │
        │ MetricsWriter consumes (after batching 5s / 10k samples, Remote Write)
        │ observedAt ← Kafka LogAppendTime; or fall back to the observed_timestamp field
        ▼
  Redis (snapshot cache: amp:metrics:snapshot:{aic})           ← latest state (preferred by snapshots/query)
  VictoriaMetrics (localhost:8428, Remote Write)               ← time-series source of truth (series/rankings/slo/capacity)
        │
        │ Query API
        ▼
  POST /acps-amp-v1/metrics/snapshots/query    ← latest snapshots (Redis first, TSDB fallback)
  POST /acps-amp-v1/metrics/series/query       ← time-series data (VictoriaMetrics)
  POST /acps-amp-v1/metrics/rankings/query     ← TopN ranking (VictoriaMetrics instant query)
  POST /acps-amp-v1/metrics/slo/evaluate       ← batch SLO evaluation (VictoriaMetrics)
  POST /acps-amp-v1/metrics/capacity/saturation← capacity saturation (VictoriaMetrics)
```

### 1.2 Port Overview

| Component | Port | Description |
|------|------|------|
| acps-infra Redpanda (Kafka) | 19092 | Metric event stream (host access) |
| acps-infra Redis | 6379 | Snapshot cache + dedup + freshness watermark |
| acps-infra VictoriaMetrics | 8428 | Remote Write ingestion + PromQL queries |
| demo-leader | 9031 | Leader business API (the metrics background task starts in the same process) |
| demo-partner | 9021-9025 | 5 Partner Agent instances (each with its own independent metrics background task) |
| monitor-server Query API | 9009 | Metrics query API (5 endpoints) |

### 1.3 Kafka Topics Involved

| Topic | Partitions | Key configuration | Purpose |
|------|------|---------|------|
| `amp.metrics` | **1** | **`message.timestamp.type=LogAppendTime`** | Metric input stream (consumed by Writer) |
| `amp.metrics.dlq` | 1 | retention 7d | Dead-letter queue for bad messages (missing timestamp, corrupted JSON, etc.) |

> `amp.metrics` must be 1 partition + `LogAppendTime`, created idempotently by
> `acps-infra/dev-infra/dev-infra.sh`. If it was created or reset manually, be sure to confirm it (see §5.1).

### 1.4 Differences from the Heartbeat Pipeline

| Dimension | Heartbeat | Metrics |
|------|-----------|---------|
| Emitter | `HeartbeatEmitter` (lightweight, only uptimeSeconds) | `MetricsEmitter` (full load/window metrics + `resource` labels) |
| Time-series storage | None (pure Redis) | VictoriaMetrics (Remote Write Prometheus wire format) |
| Source of truth | Redis | VictoriaMetrics (time series) + Redis (latest snapshot) |
| Write mode | Record by record | Batch 5s or 10k samples, then a single Remote Write |
| observedAt source | Kafka LogAppendTime | Kafka LogAppendTime (preferred); `observed_timestamp` field fallback |
| Query endpoint count | 4 (liveness / summary / query / silence/top) | 5 (snapshots / series / rankings / slo / capacity) |
| Sync plane | Yes (sync/info + sync/snapshot) | **No** |
| VictoriaMetrics dependency | Not required | Required (series/rankings/slo/capacity all go through the TSDB) |

## 2. Generating Metrics Data

### 2.1 Automatic Generation (Default, Method A: Periodic demo Emission)

After demo-leader starts, `start_metrics()` automatically emits one metrics log in the background **every 30s**:

```bash
cd demo-leader
just dev start        # includes the Heartbeat + Metrics background tasks

# Check the logs to confirm the Metrics background task has started
just dev logs | grep "AMP metrics started"
# Expect: AMP metrics started (aic=..., interval=30s)
```

Each demo-partner Agent also emits independently (**the metrics task starts only after a restart following a code update**):

```bash
cd demo-partner
just dev restart       # or stop + start
just dev logs | grep "AMP metrics started"
```

### 2.2 Inspecting the Local Files Already Written

```bash
# demo-leader (single file)
tail -1 demo-leader/logs/amp_metrics.jsonl | python3 -m json.tool
# {
#   "schema_version": "1.0",
#   "log_type": "metrics",
#   "timestamp": "2026-06-13T...",
#   "aic": "...",
#   "body": {"uptimeSeconds": 45.12, "loadMetrics": {...}, "windowMetrics": [...]},
#   "resource": {"service.name": "demo-leader", "service.namespace": "acps-demo", ...}
# }

# demo-partner (one file per Agent)
for f in demo-partner/logs/amp_metrics_*.jsonl; do
  echo "== ${f} =="; tail -1 "${f}" | python3 -m json.tool
done
```

### 2.3 Alternative: Send One Manually (Method B: SDK Inline, No Running demo Needed)

```bash
cd demo-leader
uv run python - <<'EOF'
import sys
sys.path.insert(0, '../acps-sdk')
from pathlib import Path
from acps_sdk.amp import MetricsEmitter
from acps_sdk.amp.metrics_demo import DemoMetricsSampler

aic = '1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ'
sampler = DemoMetricsSampler(aic=aic)
e = MetricsEmitter(
    Path('logs/amp_metrics.jsonl'),
    aic=aic,
    sampler=sampler,
    resource={"service.name": "demo-leader", "service.namespace": "acps-demo",
              "deployment.environment.name": "dev"},
)
log_id = e.emit_sync()
print('emitted log_id:', log_id)
EOF
```

## 3. Verifying Each Stage

### 3.1 Verifying Kafka (Fluent Bit Has Forwarded)

**Prerequisite**: confirm that `amp.metrics` is `LogAppendTime` (see [dev-runbook_en.md §3.1](./dev-runbook_en.md)).

```bash
docker exec dev-redpanda rpk topic describe amp.metrics -c | grep -i timestamp
# Expect: message.timestamp.type  LogAppendTime
```
# HIGH-WATERMARK should increase as metrics logs are written
docker exec dev-redpanda rpk topic describe amp.metrics -p
# Expect: HIGH-WATERMARK > 0 and increasing

# Consume the latest record to confirm the content
HW=$(docker exec dev-redpanda rpk topic describe amp.metrics -p | awk '$1=="0"{print $6}')
docker exec dev-redpanda rpk topic consume amp.metrics --partitions=0 --offset="$((HW-1))" --num=1 \
  | python3 -c "import sys,json; d=json.load(sys.stdin); v=json.loads(d['value']); print(json.dumps(v, indent=2, ensure_ascii=False))"
# Expect: log_type == "metrics", includes body.uptimeSeconds, includes the resource field (service.name, etc.)

# The DLQ should not grow during this verification (an existing historical watermark is acceptable, but there should be no new entries)
docker exec dev-redpanda rpk topic describe amp.metrics.dlq -p
```

> If the DLQ watermark **keeps growing**, the most common root cause is that `amp.metrics` does not have `LogAppendTime` set (see §5.1).

### 3.2 Verifying Consumer Group LAG

```bash
docker exec dev-redpanda rpk group describe monitor-server.metrics.writer.v1
# Expect: LAG approaching 0 (MetricsWriter has consumed all messages)
```

### 3.3 Verifying VictoriaMetrics (Time-Series Source of Truth)

```bash
# Health check
curl -s http://localhost:8428/health
# Expect: Alive

# Instant query: confirm metric samples have been written (replace the placeholder with the actual aic)
AIC="1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ"
curl -s "http://localhost:8428/api/v1/query?query=amp_load_uptime_seconds%7Baic%3D%22${AIC}%22%7D" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status'], len(d['data']['result']), 'series')"
# Expect: success N series (N >= 1)

# You can open vmui in a browser for interactive queries
# http://localhost:8428/vmui/
```

### 3.4 Verifying the Redis Snapshot Cache

```bash
# ZSet index (member=aic, score=observed_at_ms)
redis-cli -n 2 zrange 'amp:metrics:snapshot:index' 0 -1 withscores | head -20

# Per-AIC Hash (latest snapshot fields)
AIC="1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ"
redis-cli -n 2 hgetall "amp:metrics:snapshot:${AIC}"
# Expected fields: observed_at  uptime_seconds  load_metrics_json  window_metrics_json
#            service_name  service_namespace  deployment_env

# freshness watermark (dataFreshnessAt, used for the 503 guard)
redis-cli -n 2 get 'amp:metrics:data_freshness_at_ms'
# Expect: a millisecond timestamp (non-empty)
```

### 3.5 Verifying the Query API (5 Endpoints)

All the curl commands below are fine to run independently (they do not depend on demo); you only need infra + monitor-server online.
Replace the placeholder with the actual AIC, or omit the filter to query everything.

#### snapshots/query — Latest Snapshots (Redis First)

```bash
# Query the latest snapshot of every Agent (first 5 records)
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/snapshots/query \
  -H "Content-Type: application/json" \
  -d '{"page":{"limit":5}}' | python3 -m json.tool
# Expect: an items list, each entry containing aic, observedAt, uptimeSeconds, loadMetrics, windowMetrics

# Exact query by AIC
AIC="1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ"
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/snapshots/query \
  -H "Content-Type: application/json" \
  -d "{\"filter\":{\"conditions\":[{\"field\":\"aic\",\"op\":\"eq\",\"value\":\"${AIC}\"}]},\"page\":{\"limit\":1}}" \
  | python3 -m json.tool

# Query by service_name (the resource of demo-leader)
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/snapshots/query \
  -H "Content-Type: application/json" \
  -d '{"filter":{"conditions":[{"field":"service_name","op":"eq","value":"demo-leader"}]},"page":{"limit":5}}' \
  | python3 -m json.tool
```

#### series/query — Time-Series Data (VictoriaMetrics)

```bash
# uptimeSeconds trend over the last hour (grouped by AIC)
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
HOUR_AGO=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/series/query \
  -H "Content-Type: application/json" \
  -d "{
    \"metric\": \"uptimeSeconds\",
    \"timeRange\": {\"startAt\": \"${HOUR_AGO}\", \"endAt\": \"${NOW}\"},
    \"groupByAic\": true
  }" | python3 -m json.tool
# Expect: an items list, each entry containing metric, labels (including aic), points (timestamp + value)

# Available metric names (camelCase public names, see the §7 cross-reference table):
#   uptimeSeconds  activeTasks  queuedTasks  cpuUsage  memoryUsage
#   successRate  requestTotal  requestPerSecond  p50LatencyMs  p95LatencyMs  p99LatencyMs
```

#### rankings/query — TopN Ranking

```bash
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
HOUR_AGO=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/rankings/query \
  -H "Content-Type: application/json" \
  -d "{
    \"metric\": \"uptimeSeconds\",
    \"timeRange\": {\"startAt\": \"${HOUR_AGO}\", \"endAt\": \"${NOW}\"},
    \"aggregation\": \"latest\",
    \"direction\": \"desc\",
    \"topN\": 5
  }" | python3 -m json.tool
# Expect: an items list sorted by uptimeSeconds descending, containing aic, value, evaluatedAt
```

#### slo/evaluate — Batch SLO Evaluation

```bash
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
HOUR_AGO=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/slo/evaluate \
  -H "Content-Type: application/json" \
  -d "{
    \"timeRange\": {\"startAt\": \"${HOUR_AGO}\", \"endAt\": \"${NOW}\"},
    \"rules\": [{\"sli\": \"success_rate\", \"window\": \"PT5M\", \"target\": 99.0}]
  }" | python3 -m json.tool
# Expect: items (the evaluation result per AIC×window, containing meets=true/false, actual, target)
#        summary (total / meetsCount / breachCount)
#
# Note: sli values must be snake_case: success_rate / p95_latency_ms / p99_latency_ms / avg_latency_ms
```

#### capacity/saturation — Capacity Saturation

```bash
curl -s -X POST http://localhost:9009/acps-amp-v1/metrics/capacity/saturation \
  -H "Content-Type: application/json" \
  -d '{"activeRatioThreshold": 0.8, "queueRatioThreshold": 0.8}' \
  | python3 -m json.tool
# Expect: items, containing aic, activeRatio, queueRatio, activeTasks, maxActiveTasks, etc.
# Note: demo load is usually low, so empty items at a threshold of 0.8 is normal; you can lower it to 0.1 to verify endpoint availability
```

## 4. Quick Verification Scripts

### 4.1 smoke_metrics.py (Shortest Path, Independent of demo)

`scripts/smoke_metrics.py` connects directly to Kafka, delivers one metrics message, and polls
`snapshots/query` within 20s to confirm the snapshot is visible, while also verifying that the freshness watermark has advanced.
It only requires **infra (kafka+redis) + monitor-server** online, skipping Fluent Bit and demo.

```bash
cd monitor-server
APP_ENV=development uv run python scripts/smoke_metrics.py
```

Successful output:

```text
=== Metrics Dev Smoke ===
[INFO] monitor: http://localhost:9009, topic: amp.metrics, aic: urn:test:metrics:e2e
[OK] metrics message delivered: aic=urn:test:metrics:e2e, log_id=...
[OK] snapshot visible: aic=urn:test:metrics:e2e, uptimeSeconds=42.0
[OK] freshness watermark advanced: ... ms

[PASS] Metrics message→Redis snapshot→Query API path verified!
```

### 4.2 demo_metrics.sh (Full Pipeline, Requires demo Online)

`scripts/demo_metrics.sh` assumes all services are started and demo is periodically emitting metrics logs,
waits 40s for propagation, then asserts that `snapshots/query` has data and checks the DLQ and VictoriaMetrics.

```bash
# Recommended: bash monitor-server/scripts/demo_metrics.sh (works from any current directory; the script resolves the acps/ root automatically)
bash monitor-server/scripts/demo_metrics.sh
```

Successful output (after waiting about 40s):

```text
=== AMP Metrics Dev Demo ===
[1] Service health check...
    ✓ monitor-server status=ok, victoria_metrics=ok
[2] Waiting 40s...
[3] Local metrics file check...
    ✓ demo-leader metrics file exists (... records)
    ✓ demo-partner metrics file exists (5 Agents)
[4] Query API: snapshots/query (no filter, first 10 records)...
    returned items=6, total=6
...
✓ PASS: snapshots/query returned 6 records (total=6), Metrics path verified!
```

## 5. Troubleshooting

### 5.1 DLQ Keeps Growing (Most Common)

**Symptom**: the `amp.metrics.dlq` watermark keeps growing during this verification; the monitor-server logs contain
`UntimedMetricsError` (aic=..., log_id=...).

**Root cause**: the `amp.metrics` topic is not `LogAppendTime`, and the message body does not carry the `observed_timestamp` field.
Writer's priority order for `observedAt`: Kafka LogAppendTime (`timestamp_type=1`) → `observed_timestamp` field → DLQ.
Fluent Bit forwards the file content directly and does not inject `observed_timestamp`, so this **must** rely on LogAppendTime.

**Fix**:

```bash
# Confirm the topic configuration
docker exec dev-redpanda rpk topic describe amp.metrics -c | grep -i timestamp
# Expect: message.timestamp.type  LogAppendTime

# If it is not LogAppendTime, change it or recreate the topic:
docker exec dev-redpanda rpk topic alter-config amp.metrics --set message.timestamp.type=LogAppendTime
# Or have dev-infra recreate it per spec (this clears data):
just -f monitor-server/Justfile infra reset kafka
```

### 5.2 VictoriaMetrics Unreachable (series/rankings/slo/capacity Return 503 or degraded)

**Symptom**: `/health` returns `"victoria_metrics":"degraded"`; series-type endpoints return 503 or time out.
snapshots/query (the Redis path) **is still available** and unaffected.

**Fix**:

```bash
# Confirm container status
just -f monitor-server/Justfile infra status
# dev-victoria-metrics should show running + healthy

# If unhealthy, restart it:
docker compose -f acps-infra/dev-infra/compose.yml --profile victoria-metrics restart dev-victoria-metrics
# Or:
just -f monitor-server/Justfile infra up victoria-metrics
```

### 5.3 Redis Not Started / snapshots/query Returns 503

**Symptom**: `snapshots/query` returns `503 AMP_READ_MODEL_LAGGING`; Redis is unreachable.

```bash
redis-cli -n 2 ping        # Expect PONG; if it times out, Redis is not started
just -f monitor-server/Justfile infra up redis
just -f monitor-server/Justfile infra wait redis
```

> `503 AMP_READ_MODEL_LAGGING` can also appear right after monitor-server starts (the freshness watermark is not established yet),
> and recovers automatically once the first batch of messages has been processed by MetricsWriter (usually within 5s).

### 5.4 snapshots/query Returns Empty but series/query Has Data

**Symptom**: `snapshots/query` returns `items=[]`, but VictoriaMetrics already has samples (series has data).

**Root cause**: the Redis snapshot TTL has expired, or Writer batch processing has not finished yet (snapshots are written after a successful Remote Write).

**Diagnosis**:

```bash
# Confirm the Redis index has entries
redis-cli -n 2 zcard 'amp:metrics:snapshot:index'
# If it is 0, the snapshot has expired or was never written

# Check the monitor-server logs to confirm Remote Write succeeded
just -f monitor-server/Justfile app logs | grep -E "remote_write|snapshot_cache"
```

### 5.5 No metrics Enter Kafka After Fluent Bit Starts

**Symptom**: the `amp.metrics` watermark does not grow, while amp.audit / amp.heartbeat are normal.

**Diagnosis**:

1. Confirm that the Fluent Bit stdout has `[output:kafka:kafka.2] worker #0 started` (the third OUTPUT).
2. Check whether `fluent-bit.conf` contains the metrics INPUT/OUTPUT sections (`tag=amp.metrics`).
3. Confirm that `demo-leader/logs/amp_metrics.jsonl` exists and is growing:
   ```bash
   ls -la demo-leader/logs/amp_metrics.jsonl
   wc -l demo-leader/logs/amp_metrics.jsonl
   ```
4. If the path in the conf does not exist (a hard-coded absolute path issue), Fluent Bit skips that INPUT without reporting an error:
   Confirm that the path in `config/fluent-bit/fluent-bit.conf` matches the actual path on this machine.

See the general Fluent Bit troubleshooting in [dev-runbook_en.md §5](./dev-runbook_en.md).

## 6. Relationship to Automated Tests

| Layer | Entry point | Coverage | Requires demo |
|------|------|----------|--------------|
| Manual full pipeline | §3 of this document | Fluent Bit + real demo periodic emission + all 5 endpoints | Yes |
| Quick path script | `scripts/smoke_metrics.py` | Kafka → Writer → Redis → snapshots/query | No |
| One-command full pipeline | `scripts/demo_metrics.sh` | All services online, asserts snapshots/query has data | Yes |
| pytest e2e | `just test e2e -k metrics` (`tests/e2e/test_metrics_*`) | Black box: ingest / query / retention, 9 cases | No |
| pytest integration | `just test integration -k metrics` | ASGI + respx mock, each endpoint verified in isolation | No |
| pytest unit | `just test unit -k metrics` | Pure functions: series mapping / freshness / cursor / encode | No |

> The automated E2E cases (`tests/e2e/`) complement this manual: pytest covers "protocol correctness", while this manual covers
> the full-pipeline integration scenario of "multiple projects + real Fluent Bit + periodic demo emission".

## 7. Metric Naming Cross-Reference

The Query API accepts camelCase **public names**; internal Prometheus series names are not exposed externally.

| Public name (API input) | Internal Prometheus name | Description |
|---------------------|--------------------------|------|
| `uptimeSeconds` | `amp_load_uptime_seconds` | System uptime (seconds) |
| `activeTasks` | `amp_load_active_tasks` | Number of tasks currently executing |
| `queuedTasks` | `amp_load_queued_tasks` | Number of tasks waiting in the queue |
| `maxActiveTasks` | `amp_load_max_active_tasks` | Maximum number of concurrent tasks |
| `maxQueuedTasks` | `amp_load_max_queued_tasks` | Maximum queue capacity |
| `cpuUsage` | `amp_load_cpu_usage` | CPU usage (0–100%) |
| `memoryUsage` | `amp_load_memory_usage` | Memory usage (0–100%) |
| `diskUsage` | `amp_load_disk_usage` | Disk usage (0–100%) |
| `successRate` | `amp_window_success_rate` | Sliding-window success rate (%) |
| `requestTotal` | `amp_window_request_total` | Total requests in the window |
| `requestPerSecond` | `amp_window_request_per_second` | Requests per second |
| `avgLatencyMs` | `amp_window_avg_latency_ms` | Average latency (milliseconds) |
| `p50LatencyMs` | `amp_window_latency_ms` (quantile=p50) | P50 latency (milliseconds) |
| `p95LatencyMs` | `amp_window_latency_ms` (quantile=p95) | P95 latency (milliseconds) |
| `p99LatencyMs` | `amp_window_latency_ms` (quantile=p99) | P99 latency (milliseconds) |
