**[English](dev-runbook-access_en.md) | [中文](dev-runbook-access.md)**

# Development-Mode Access Joint Verification

**Prerequisite**: first follow [dev-runbook_en.md](./dev-runbook_en.md) to complete the service startup (monitor-server, demo-leader,
demo-partner, Fluent Bit, ClickHouse). This document only covers the verification content for the Access pipeline in development mode.

**Quick checks before starting verification** (if any one is not satisfied, fix it first and then continue):

```bash
# 1. amp.access must be LogAppendTime, partition count ≥ 4
docker exec dev-redpanda rpk topic describe amp.access -c | grep timestamp.type
docker exec dev-redpanda rpk topic describe amp.access -p
# Expected: LogAppendTime; PARTITION row count ≥ 4 (when there is only 1 partition, run `just infra up kafka` from §3.1 of dev-runbook_en.md)

# 2. Fluent Bit must contain the kafka.3 worker (access OUTPUT)
# After startup/restart, stdout should show: [output:kafka:kafka.3] worker #0 started

# 3. ClickHouse reachable
curl -s http://localhost:8123/ping
# Expected: Ok.

# 4. After demo code or acps-sdk is updated, demo-leader / demo-partner must be restarted (Access is event-driven and needs real business requests)
#    If demo-partner fails at startup with cannot import name 'AccessEmitter', first confirm that acps-sdk exports AccessEmitter and run uv sync
```

Compared with the Heartbeat / Metrics pipelines, Access has three fundamental differences:

- **Event-driven**: a business request must be triggered (leader `/api/v1/submit` → partner `/rpc`) before there are logs; being online does not emit periodically on its own.
- **Bilateral emission**: leader emits the caller client span (SDK `AipRpcClient`), partner emits the callee server span (`/rpc` endpoint).
- **Source of truth ClickHouse**: 7 Query endpoints; trace/topology rely on `traceparent` propagation.
- **Non-emission boundaries (O-A1/O-A4)**: user→leader inbound, group `/group/rpc`, and partner→leader callbacks do not emit.

On a first read, we suggest using the [Chapter 4 quick scripts](#4-quick-verification-scripts) to confirm the end-to-end path,
then following [Chapters 2–3](#2-generating-access-data) to run the full demo → Fluent Bit → Kafka → Writer → ClickHouse → API pipeline.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
User → demo-leader /api/v1/submit (no emission, O-A1)
  → AipRpcClient._send_request (caller span → logs/amp_access.jsonl)
  → demo-partner POST /rpc (server span → logs/amp_access_*.jsonl)
  → Fluent Bit kafka.3 → Kafka amp.access
  → AccessWriter → ClickHouse + Redis
  → Query API (7 endpoints)
```

| Component | Port | Description |
|------|------|------|
| ClickHouse | 8123 / 19010 | Access source of truth |
| Kafka amp.access | 19092 | ≥4 partitions, LogAppendTime |
| Redis | 6379 | Deduplication + watermark + trace hint |
| demo-leader | 9031 | caller access logs |
| demo-partner | 9021-9025 | callee access logs (one file per Agent) |
| monitor-server | 9009 | Access Query API |

### 1.3 Kafka Topics Involved

| Topic | Partitions | Key Configuration | Purpose |
|------|------|---------|------|
| `amp.access` | **≥4** | **`message.timestamp.type=LogAppendTime`** | Access input stream (consumed by Writer) |
| `amp.access.dlq` | 1 | retention 7d | Bad-message dead letter |

### 1.4 Differences from the Heartbeat Pipeline

| Dimension | Heartbeat | Access |
|------|-----------|--------|
| Trigger mode | Emits periodically as long as the process is online | **Every business RPC interaction** |
| Source of truth | Redis | ClickHouse |
| Tracing | None | trace_id/span_id + traceparent propagation |
| Query endpoints | 4 + Sync | 7 (events/operations/errors/slow/traces/topology) |
| Bilateral emission | No | Yes (caller SDK + callee `/rpc`) |

## 2. Generating Access Data

### 2.1 Triggering a Business Request (Recommended)

The following commands must be run from the **`acps/` repository root** (consistent with `dev-runbook_en.md` §3.5 Fluent Bit).

**Recommended (bilateral `/rpc` instrumentation)**: use a hotel-type request that hits only the **RPC Partner** (`streaming=false`, such as `china_hotel`), so that the planner does not also select `china_transport` (`streaming=true`) and go through `/stream` — the current design emits Access only in `AipRpcClient` (`/rpc`) and partner `/rpc`, and the **`/stream` path does not emit**.

```bash
curl -s -X POST http://localhost:9031/api/v1/submit \
  -H "Content-Type: application/json" \
  -d '{"query":"I want to book a hotel in Beijing for three nights, budget 500 yuan per night, 2 guests","clientRequestId":"access-dev-1","mode":"direct_rpc"}'
```

A multi-dimensional itinerary (such as "three-day Beijing tour") may schedule hotel (RPC) + transport (stream) concurrently; the leader-side caller span will still exist, but the **partner-side `amp_access_*.jsonl` may be empty** (stream is not instrumented). For full-pipeline bilateral verification, use the hotel request above, or see the [§4.2](#42-e2e_access_demosh) script (it only asserts that the Query API is reachable).

```bash
# Generic example (may go through stream, see the note above)
curl -s -X POST http://localhost:9031/api/v1/submit \
  -H "Content-Type: application/json" \
  -d '{"query":"Help me plan a three-day Beijing tour","clientRequestId":"access-dev-1","mode":"direct_rpc"}'
```

After submitting, wait about 30–60s (LLM planning + Partner RPC polling complete), then run §2.2 / §3.

### 2.2 Inspecting Local Files

Run from the **`acps/` root directory**:

```bash
tail -1 demo-leader/logs/amp_access.jsonl | python3 -m json.tool
# partner: one file per Agent, named after the partners/online/<agent_dir> directory name, for example:
tail -1 demo-partner/logs/amp_access_china_hotel.jsonl | python3 -m json.tool
# or list all of them:
ls demo-partner/logs/amp_access_*.jsonl
```

Expected: `log_type` is `access`, containing `trace_id`, `span_id`, `caller`, `callee`; it does **not** contain `integrity`. At least one line each for leader and partner (caller + server span under the same `trace_id`).

### 2.3 Fallback: Produce One Message Directly to Kafka (No demo Needed)

See [§4.1](#41-smoke_accesspy--e2e_access_verifypy).

## 3. Stage-by-Stage Verification

### 3.1 Kafka

```bash
docker exec dev-redpanda rpk topic describe amp.access -p
docker exec dev-redpanda rpk topic describe amp.access.dlq -p
# amp.access is expected to have ≥ 4 PARTITIONs; when there is only 1 partition: just -f monitor-server/Justfile infra up kafka
```

The DLQ should not keep growing with normal requests.

### 3.2 Consumer Group

```bash
docker exec dev-redpanda rpk group describe monitor-server.access.writer.v1
```

LAG should approach 0.

### 3.3 ClickHouse

```bash
curl -s 'http://localhost:8123/?query=SELECT%20count()%20FROM%20amp.access_events'
curl -s 'http://localhost:8123/?query=SELECT%20count()%20FROM%20amp.access_topology_edge_5m'
```

### 3.4 Redis

```bash
redis-cli -n 2 --scan --pattern 'amp:access:*' | head
```

Expected: keys such as the deduplication key, partition watermarks, and trace hints exist.

### 3.5 Query API (Seven Endpoints)

```bash
AIC=<the aic of some online partner>
TRACE=<take trace_id from events or the local jsonl>
START=$(date -u -v-15M +%Y-%m-%dT%H:%M:%SZ); END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
BASE=http://localhost:9009/acps-amp-v1/access

# events
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"page\":{\"limit\":20}}" | python3 -m json.tool

# operations
curl -s -X POST "$BASE/operations/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"groupBy\":[\"endpoint\"]}" | python3 -m json.tool

# errors/attribution (requires analyticsEnabled)
curl -s -X POST "$BASE/errors/attribution" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"groupBy\":[\"endpoint\"],\"topN\":10}" | python3 -m json.tool

# slow-requests/top (requires analyticsEnabled)
curl -s -X POST "$BASE/slow-requests/top" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"topN\":10}" | python3 -m json.tool

# traces/query (requires apmEnabled)
curl -s -X POST "$BASE/traces/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"page\":{\"limit\":20}}" | python3 -m json.tool

# traces/{traceId} (the response headers include AMP-Data-Freshness-At; use -I to view it optionally)
curl -s "$BASE/traces/$TRACE" | python3 -m json.tool
curl -sI "$BASE/traces/$TRACE" | grep -i '^amp-'

# topology/query (requires apmEnabled; only the callee inbound perspective, single-sided counting)
curl -s -X POST "$BASE/topology/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"groupBy\":\"aic\"}" | python3 -m json.tool
```

> For the Profile switches, see `analytics_enabled` / `apm_enabled` under `[access]` in `config/development.toml`.

## 4. Quick Verification Scripts

### 4.1 `smoke_access.py` / `e2e_access_verify.py`

Requires only infra + ClickHouse + monitor-server (no dependency on demo / Fluent Bit):

```bash
cd monitor-server
APP_ENV=development uv run python scripts/smoke_access.py
# or: uv run python scripts/e2e_access_verify.py
```

### 4.2 `e2e_access_demo.sh`

With all services online, trigger `/submit` and then poll the Query API:

```bash
# Recommended: bash monitor-server/scripts/e2e_access_demo.sh (any current directory works, it only calls the HTTP API)
bash monitor-server/scripts/e2e_access_demo.sh
```

## 5. Troubleshooting

| Symptom | Root Cause | Remedy |
|------|------|------|
| All access logs go to the DLQ | topic is not LogAppendTime | `just -f monitor-server/Justfile infra up kafka` |
| No access logs at all | No business request was triggered (no periodic emission) | Send `/api/v1/submit` (the §2.1 hotel RPC request is recommended) |
| partner has no `amp_access_*.jsonl` | It went through `/stream`, or demo-partner was not restarted | Use the §2.1 hotel request; `just -f demo-partner/Justfile app restart` |
| demo-partner startup ImportError | acps-sdk does not export `AccessEmitter` | Upgrade/sync acps-sdk, then restart partner |
| traces has only one span layer | traceparent was not propagated | Confirm the leader injects the emitter; partner `/rpc` reads the header |
| topology has no edges / is doubled | Only the caller emitted, or callee.aic is filled in incorrectly | Confirm the partner server span; the MV direction convergence counts only callee rows |
| Query 503 `AMP_READ_MODEL_LAGGING` | CH unreachable, high consumption LAG, or **stale Redis partition watermarks** | `curl :8123/ping`; `rpk group describe monitor-server.access.writer.v1`; see §5.1 below |
| events/query returns 503 but CH already has data | `amp:access:wm:partitions` contains Kafka partitions that no longer exist, so min(wm) is too old | Clean up per §5.1, or restore 4 partitions with `just infra up kafka` and then `just dev restart` |
| analytics/apm 404 | Profile not enabled | Enable the corresponding switch in `[access]` and restart |
| Fluent Bit has no access | Missing kafka.3 / wrong path | Confirm `kafka.3 worker #0 started` |

### 5.1 Query 503: Access Read-Model Watermark (Redis DB 2)

Access freshness keys live in **Redis logical database 2** (`redis-cli -n 2`). `events/query` returns 503 when the overall watermark lags by more than 5 minutes.

**Common scenario: after a Docker/Kafka reset**, `amp:access:wm:partitions` still registers partitions `1,2,3`, but `amp.access` has shrunk to 1 partition,
so the Writer no longer updates the watermarks for 1–3 → `min(wm)` freezes → permanent 503.

```bash
# View the registered partitions and watermarks (monitor dev database 2)
redis-cli -n 2 smembers amp:access:wm:partitions
redis-cli -n 2 mget amp:access:wm:0 amp:access:wm:1 amp:access:wm:2 amp:access:wm:3

# Remedy A: restore 4 Kafka partitions (recommended, consistent with dev-infra)
just -f monitor-server/Justfile infra up kafka
just -f monitor-server/Justfile app restart

# Remedy B: only clean up the stale partition watermarks (emergency for the single-partition development state)
# Replace <ids> with the members from smembers that exceed the current topic partition count, for example 1 2 3
redis-cli -n 2 del amp:access:wm:1 amp:access:wm:2 amp:access:wm:3
redis-cli -n 2 srem amp:access:wm:partitions 1 2 3
just -f monitor-server/Justfile app restart
```

## 6. Relationship to Automated Tests

| Layer | Entry Point |
|------|------|
| Manual integration testing | This document |
| Smoke scripts | `scripts/smoke_access.py` |
| Full-pipeline demo | `scripts/e2e_access_demo.sh` |
| pytest E2E | `just test e2e -k access` |

Coverage: `test_access_ingest_flow`, `test_access_apm_flow`, `test_access_analytics_flow`, `test_access_bilateral_flow`.
