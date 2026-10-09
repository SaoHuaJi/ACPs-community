**[English](dev-runbook-system_en.md) | [中文](dev-runbook-system.md)**

# Development-Mode System Joint Verification

**Prerequisite**: first follow [dev-runbook_en.md](./dev-runbook_en.md) to complete the service startup (monitor-server, demo-leader,
demo-partner, Fluent Bit, OpenSearch). This document only covers the verification content for the System pipeline in development mode.

**Quick checks before starting verification** (if any one is not satisfied, fix it first and then continue):

```bash
# 1. amp.system must be LogAppendTime
docker exec dev-redpanda rpk topic describe amp.system -c | grep timestamp.type
# Expected: LogAppendTime

# 2. Fluent Bit must contain the kafka.5 worker (system OUTPUT)
# After startup/restart, stdout should show: [output:kafka:kafka.5] worker #0 started

# 3. OpenSearch reachable
curl -s 'http://localhost:9200/_cluster/health?pretty'
# Expected: status is green or yellow

# 4. After demo code, acps-sdk, or the monitor-server System module is updated, restart (from the acps/ root directory):
cd demo-leader && just dev restart && cd ..
cd demo-partner && just dev restart && cd ..
cd monitor-server && just dev restart && cd ..
# System is event-driven; apart from lifecycle, real business requests are required to produce logs
```

Compared with Heartbeat / Metrics, System has three fundamental differences:

- **Event-driven**: a business request must be triggered (leader `/api/v1/submit` → partner internal LLM/Skill) before there are logs.
- **Single-sided emission**: partner and leader each write local NDJSON, with no traceparent propagation; they are correlated using `correlation_id=task_id`.
- **Source of truth OpenSearch**: a single Query endpoint `POST /system/events/query`; severity is the core filtering dimension.

On a first read, we suggest using the [Chapter 4 quick scripts](#4-quick-verification-scripts) to confirm the end-to-end path,
then following [Chapters 2–3](#2-generating-system-event-data) to run the full demo → Fluent Bit → Kafka → Writer → OpenSearch → API pipeline.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
User → demo-leader /api/v1/submit
  → demo-partner GenericRunner._call_llm / _execute_skill (→ logs/amp_system_*.jsonl)
  → demo-leader executor errors/timeouts (→ logs/amp_system.jsonl)
  → Fluent Bit kafka.5 → Kafka amp.system
  → SystemWriter → OpenSearch amp-system-events-*
  → Query API POST /system/events/query
```

| Component | Port | Description |
|------|------|------|
| OpenSearch | 9200 | System source of truth |
| Kafka amp.system | 19092 | ≥4 partitions, LogAppendTime |
| Redis | 6379 | Deduplication + watermark |
| demo-leader | 9031 | Orchestration errors/timeouts + lifecycle |
| demo-partner | 9021-9025 | LLM/Skill/capacity/lifecycle (one file per Agent) |
| monitor-server | 9009 | System Query API |

### 1.2 Kafka Topics Involved

| Topic | Partitions | Key Configuration | Purpose |
|------|------|---------|------|
| `amp.system` | **≥4** | **`message.timestamp.type=LogAppendTime`** | System event input stream |
| `amp.system.dlq` | 1 | retention 7d | Bad-message dead letter |

## 2. Generating System Event Data

### 2.1 Triggering a Business Request (Recommended)

```bash
curl -X POST http://localhost:9031/api/v1/submit \
  -H 'Content-Type: application/json' \
  -d '{"query":"I want to book a hotel in Beijing for three nights, budget 500 yuan per night, 2 guests","clientRequestId":"system-demo-001","mode":"direct_rpc"}'
```

The partner-side LLM event's `correlation_id` looks like `{activeTaskId}:{partner_aic}` (not the `activeTaskId` alone returned by the leader). The full-pipeline script parses the complete value from `amp_system_*.jsonl` and then queries the Query API.

### 2.2 Inspecting Local NDJSON Files

The following commands are run from the **acps/ root directory**. `tail` inserts `==> file <==` separator lines for multiple globbed files and cannot be piped directly into `json.tool`; instead inspect the last line of a single file (the §2.1 hotel request mainly writes to `china_hotel`):

```bash
tail -1 demo-partner/logs/amp_system_china_hotel.jsonl | python3 -m json.tool
tail -1 demo-leader/logs/amp_system.jsonl | python3 -m json.tool
```

Expect to see `body.category` as `llm`, `skill`, `lifecycle`, and so on, and `severity_number` as 9/13/17.

### 2.3 Fallback: Write Directly to the leader system File to Test Fluent Bit

```bash
echo '{"schema_version":"1.0","log_type":"system","log_id":"manual-test-001","timestamp":"2026-06-14T10:00:00.000000Z","aic":"test-aic","severity_number":9,"severity_text":"INFO","body":{"message":"manual fluent bit test","category":"test"}}' \
  >> demo-leader/logs/amp_system.jsonl
```

> The timestamp must include fractional seconds (consistent with the format written by partner), otherwise the Fluent Bit `amp_audit_json` parser will warn.

## 3. Layered Verification

### 3.1 Kafka Watermark

```bash
docker exec dev-redpanda rpk topic describe amp.system -p
docker exec dev-redpanda rpk topic consume amp.system --num 1 --format json
```

### 3.2 Consumer Group LAG

```bash
docker exec dev-redpanda rpk group describe monitor-server.system.writer.v1
```

### 3.3 OpenSearch Documents

```bash
curl -s 'http://localhost:9200/amp-system-events-*/_count'
curl -s 'http://localhost:9200/amp-system-events-*/_search?q=category:llm&size=3' | python3 -m json.tool
```

### 3.4 Redis Watermark

```bash
redis-cli -n 2 --scan --pattern 'amp:system:*'
```

### 3.5 Query API (events/query)

```bash
AIC=<the aic of some online partner>
TASK_ID=<take one correlation_id from amp_system_*.jsonl (not activeTaskId alone)>
START=$(date -u -v-30M +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)   # after the §2.1 submit this must be re-run, so that endAt covers the new events
BASE=http://localhost:9009/acps-amp-v1/system

# The 20 most recent system events
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"page\":{\"limit\":20}}" \
  | python3 -m json.tool

# Filter by aic
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"aic\",\"op\":\"eq\",\"value\":\"$AIC\"}]},\"page\":{\"limit\":20}}" \
  | python3 -m json.tool

# Filter by correlationId (in §3.5 you must use the complete correlation_id from the NDJSON, not activeTaskId alone)
# timeRange.endAt is refreshed dynamically during polling, so that events written after submit do not fall outside a fixed endAt
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"correlationId\",\"op\":\"eq\",\"value\":\"$TASK_ID\"}]},\"page\":{\"limit\":50}}" \
  | python3 -m json.tool

# Filter error events by severityNumber
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"severityNumber\",\"op\":\"gte\",\"value\":17}]},\"page\":{\"limit\":20}}" \
  | python3 -m json.tool

# keyword search
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"keyword\":\"LLM call completed\",\"page\":{\"limit\":20}}" \
  | python3 -m json.tool
```

## 4. Quick Verification Scripts

### 4.1 Direct Kafka Connection (No demo Dependency)

```bash
cd monitor-server
APP_ENV=development uv run python scripts/e2e_system_verify.py
```

### 4.2 Full-Pipeline demo

The script automatically resolves the absolute path of `demo-partner/logs/amp_system_*.jsonl`, so **any current directory works**:

```bash
bash monitor-server/scripts/e2e_system_demo.sh
```

## 5. Troubleshooting

| Symptom | Possible Cause | Handling |
|------|---------|------|
| No system logs at all | System is **event-driven**; no logs are produced without a business request (except lifecycle S-P6/S-P7/S-L3/S-L4) | Restart demo to confirm the lifecycle lines, then `POST /api/v1/submit` |
| `amp.system` watermark does not increase | Fluent Bit is not started or the path is wrong | Confirm the `kafka.5` worker; check that `amp_system*.jsonl` has new lines |
| Kafka has messages but OpenSearch has no data | SystemWriter is not started / OpenSearch is unreachable | `rpk group describe monitor-server.system.writer.v1` should be Stable; `cd monitor-server && just dev restart`; `curl http://localhost:9200/_cat/indices/amp-system*` |
| `monitor-server.system.writer.v1` STATE Dead | monitor-server was started before the System module was merged, or the process has exited | `cd monitor-server && just dev restart`; confirm LAG decreases and OpenSearch `_count` grows |
| demo-partner `just dev start` port already in use | A leftover partner child process occupies 9021–9025 | `cd demo-partner && just dev restart`; look for `address already in use` in `logs/partners_base.log` |
| `events/query` keyword returns nothing | `search_text` was not generated, or body is missing `message` | Query OpenSearch directly: `/_search?q=message:*failed*` |
| `correlationId` filter returns nothing | partner `correlation_id` is `{activeTaskId}:{partner_aic}`, not the leader `activeTaskId` alone | Take the complete `correlation_id` from `amp_system_*.jsonl` and query again |
| Fluent Bit started but system does not enter Kafka | macOS is missing `Workers 1` / wrong path / missing the sixth OUTPUT | Confirm `kafka.5 worker #0 started`; verify the absolute path; delete `/tmp/fluentbit-*-system.db` and restart |
| Everything goes to the DLQ | `amp.system` topic is not LogAppendTime | `rpk topic describe amp.system -c` to confirm LogAppendTime |
| severityNumber filter does not take effect | `emit_sync` did not pass `severity_number` | Check the top-level `severityNumber` (not inside body) |

## 6. Relationship to Automated Tests

| Layer | Entry Point |
|------|------|
| Unit tests | acps-sdk / demo-partner / demo-leader `tests/unit/test_*_system*.py` |
| monitor E2E | `just test e2e -k system` |
| Smoke script | `scripts/e2e_system_verify.py` |
| Full-pipeline demo | `scripts/e2e_system_demo.sh` |
