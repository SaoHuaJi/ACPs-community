**[English](dev-runbook-message_en.md) | [中文](dev-runbook-message.md)**

# Development-Mode Message Joint Verification

**Prerequisite**: first complete service startup by following [dev-runbook_en.md](./dev-runbook_en.md) (monitor-server, demo-leader,
demo-partner, Fluent Bit, ClickHouse, RabbitMQ). **Group mode additionally requires mq-auth-server** (RabbitMQ EXTERNAL authentication and group ACL):

```bash
cd mq-auth-server
just dev bootstrap && just dev restart
# Group API :9007 requires an mTLS client certificate; the development environment uses the official probe (which loads the certs/client.pem trio by default)
APP_ENV=development uv run python -m app.core.health_probe
# Expected: exit code 0 (HTTP 200). Do not use curl -sk (without a client certificate the handshake fails; on macOS LibreSSL, even with --cert the dev private key format may be unsupported)
# dev-infra Redis maps to host port 6379; mq-auth uses DB 0 (REDIS_URL=redis://localhost:6379/0)
```

demo-leader must have **group mode** enabled. This document only covers verification of the Message pipeline in development mode.

**Quick checks before starting verification** (if any one is not satisfied, fix it first before continuing):

```bash
# 1. amp.message must be LogAppendTime, with ≥ 4 partitions
docker exec dev-redpanda rpk topic describe amp.message -c | grep -E 'timestamp.type|partition'
docker exec dev-redpanda rpk topic describe amp.message -p
# Expected: LogAppendTime; number of PARTITION lines ≥ 4 (when there is only 1 partition, run just infra up kafka from §3.1 of dev-runbook.md)

# 2. Fluent Bit must include the kafka.4 worker (message OUTPUT)
# After startup/restart, stdout should show: [output:kafka:kafka.4] worker #0 started

# 3. ClickHouse is reachable
curl -s http://localhost:8123/ping
# Expected: Ok.

# 4. demo-leader group mode is enabled, and demo-leader / demo-partner must be restarted after the demo code or acps-sdk is updated

# 5. The Message Writer consumer group must be stable (during joint verification, avoid Uvicorn hot reload interrupting background tasks)
#    set config/development.toml [server] reload = false and then just dev restart;
#    or confirm the Justfile already has a reload-exclude for logs/* (configured by default)
docker exec dev-redpanda rpk group describe monitor-server.message.writer.v1
# Expected: STATE Stable, MEMBERS ≥ 1
```

Compared with the Access / Metrics pipelines, Message has three fundamental differences:

- **Event-driven + group mode only**: logs are produced only when a group task is triggered (leader `/api/v1/submit` + `mode=group`); direct-connect RPC mode produces no message logs (that is covered by Access).
- **One message = multiple independent logs**: send / receive / ack (and abnormal nack) each produce one NDJSON line; lifecycles are merged by `messageId`.
- **ClickHouse is the source of truth**: 6 Query endpoints; `destination.name` uniformly takes the group FANOUT exchange (the queue is recorded in `subscriptionName`).

For a first read, it is recommended to first use [the quick scripts in Chapter 4](#4-quick-verification-scripts) to confirm the end-to-end path,
then follow [Chapters 2–3](#2-generating-message-data) to run the full demo → Fluent Bit → Kafka → Writer → ClickHouse → API pipeline.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
User → demo-leader /api/v1/submit (mode=group, does not emit HTTP inbound logs)
  → GroupLeaderMqClient publish (send → logs/amp_message.jsonl)
  → RabbitMQ FANOUT exchange
  → GroupPartnerMqClient consume (receive/ack → logs/amp_message_*.jsonl)
  → Fluent Bit kafka.4 → Kafka amp.message
  → MessageWriter → ClickHouse + Redis
  → Query API (6 endpoints)
```

| Component | Port | Description |
|------|------|------|
| RabbitMQ | 5671 / 15672 | Group broker |
| ClickHouse | 8123 / 19010 | Message source of truth |
| Kafka amp.message | 19092 | ≥4 partitions, LogAppendTime |
| Redis | 6379 | Deduplication + read-model watermark |
| demo-leader | 9031 | leader send message logs |
| demo-partner | 9021-9025 | partner receive/ack logs (one file per Agent) |
| monitor-server | 9009 | Message Query API |

### 1.3 Kafka Topics Involved

| Topic | Partitions | Key Configuration | Purpose |
|------|------|---------|------|
| `amp.message` | **≥4** | **`message.timestamp.type=LogAppendTime`** | Message input stream (consumed by Writer) |
| `amp.message.dlq` | 1 | retention 7d | Bad-message dead letter |

### 1.4 Differences from the Access Pipeline

| Dimension | Access | Message |
|------|--------|---------|
| Trigger | Every RPC interaction | **Group mode only** RabbitMQ broadcast |
| Logs per message | 1 RPC = 2 entries (caller+callee) | 1 task-command = 1 send + N receive + N ack |
| Tracing | traceparent HTTP propagation | traceparent AMQP header + O-M6 ContextVar |
| Query endpoints | 7 | 6 (events/lifecycles/deadletters/destinations/throughput) |

## 2. Generating Message Data

### 2.1 Trigger a Group Task (Recommended)

Run this in the **`acps/` repository root**:

```bash
curl -s -X POST http://localhost:9031/api/v1/submit \
  -H "Content-Type: application/json" \
  -d '{"query":"I want to book a hotel in Beijing for three nights, budget 500 yuan per night, 2 guests","clientRequestId":"message-dev-1","mode":"group"}'
```

After submitting, wait about 30–240s (LLM planning + Partner group processing; the `e2e_message_demo.sh` polling limit is 240s), then run §2.2 / §3.

### 2.2 Check the Local Files

```bash
tail -1 demo-leader/logs/amp_message.jsonl | python3 -m json.tool
# the partner only creates the corresponding file after successfully joining the group and consuming the exchange, for example:
tail -1 demo-partner/logs/amp_message_china_hotel.jsonl | python3 -m json.tool
ls demo-partner/logs/amp_message_*.jsonl
```

Expected: `log_type` is `message`, containing `eventType` (send/receive/ack), `messageId`, `destination`, `trace_id`. If only the leader-side `amp_message.jsonl` is present, confirm whether mq-auth-server and the group task successfully created the group (see §5 Troubleshooting).

### 2.3 Alternative: Produce a Batch Directly to Kafka (No Demo Needed)

See [§4.1](#41-e2e_message_verifypy).

## 3. Stage-by-Stage Verification

### 3.1 Kafka

```bash
docker exec dev-redpanda rpk topic describe amp.message -p
# Read from the existing watermark (-o end blocks when there are no new messages, so use -p/-o instead):
docker exec dev-redpanda rpk topic consume amp.message -p 1 -o 0 --num 3 | python3 -m json.tool
docker exec dev-redpanda rpk topic describe amp.message.dlq -p
```

Expected: the `amp.message` watermark increases with group tasks; the consumed records have `log_type=="message"`; the DLQ does not grow.

### 3.2 Consumer Group

```bash
docker exec dev-redpanda rpk group describe monitor-server.message.writer.v1
```

Expected: LAG approaches 0.

### 3.3 ClickHouse

```bash
docker exec dev-clickhouse clickhouse-client -q \
  "SELECT event_type, count() FROM amp.message_events GROUP BY event_type ORDER BY count() DESC"
```

Expected: the send/receive/ack row counts grow with tasks; the Compactor periodically recomputes `message_lifecycle`.

### 3.4 Redis

```bash
redis-cli -n 2 --scan --pattern 'amp:message:*' | head
```

Expected: the deduplication keys and read-model watermark keys exist.

### 3.5 Verify the Query API (6 Endpoints)

```bash
MID=<messageId taken from one amp.message record>
START=$(date -u -v-15M +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
BASE=http://localhost:9009/acps-amp-v1/message

# events: raw message events
curl -s -X POST "$BASE/events/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"system\",\"op\":\"eq\",\"value\":\"rabbitmq\"}]},\"page\":{\"limit\":20}}" | python3 -m json.tool

# lifecycles: merged by messageId
curl -s -X POST "$BASE/lifecycles/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"messageId\",\"op\":\"eq\",\"value\":\"$MID\"}]}}" | python3 -m json.tool

# lifecycles/{messageId}: single-message lifecycle
curl -s "$BASE/lifecycles/$MID" | python3 -m json.tool

# deadletters: dead-letter messages (must filter by a lifecycle key, such as traceId)
curl -s -X POST "$BASE/deadletters/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"filter\":{\"conditions\":[{\"field\":\"traceId\",\"op\":\"eq\",\"value\":\"<trace_id>\"}]},\"page\":{\"limit\":10}}" | python3 -m json.tool

# destinations: point-in-time destination state (the demo's default Null source → 503 STATE_SNAPSHOT_UNAVAILABLE, which is expected)
curl -s -X POST "$BASE/destinations/query" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"}}" | python3 -m json.tool

# destinations/throughput: produced/consumed trend for the group exchange (destinationName + system are top-level fields)
curl -s -X POST "$BASE/destinations/throughput" -H 'Content-Type: application/json' \
  -d "{\"timeRange\":{\"startAt\":\"$START\",\"endAt\":\"$END\"},\"destinationName\":\"<group-exchange>\",\"system\":\"rabbitmq\",\"step\":\"PT5M\"}" | python3 -m json.tool
```

> `lifecycles`/`deadletters` require `reliability_enabled=true`; `destinations/throughput` requires `destination_enabled=true` (in the `[message]` section of `config/development.toml`).
> `lifecycles/query` and `deadletters/query` must carry a selective filter (`messageId` / `traceId` / `correlationId`, or `system` + `destination.name` + `timeRange`).

## 4. Quick Verification Scripts

### 4.1 `e2e_message_verify.py`

Only infra + ClickHouse + monitor-server are needed (no dependency on demo / Fluent Bit):

```bash
cd monitor-server
APP_ENV=development uv run python scripts/e2e_message_verify.py
```

### 4.2 `e2e_message_demo.sh`

All services online + group mode:

```bash
# Recommended: bash monitor-server/scripts/e2e_message_demo.sh (works from any current directory; it only calls the HTTP API)
bash monitor-server/scripts/e2e_message_demo.sh
```

## 5. Troubleshooting

| Symptom | Root Cause | Resolution |
| --- | --- | --- |
| All message logs go into `amp.message.dlq` | `amp.message` is not LogAppendTime and emission does not carry `observedTimestamp` | `rpk topic alter-config amp.message --set message.timestamp.type=LogAppendTime` or `just infra reset kafka` |
| **No message logs at all** | Message is **produced in group mode only**; direct-connect mode or no group task | Send `/submit` with `mode=group`; confirm RabbitMQ is reachable and the partner has joined the group |
| `lifecycles/{messageId}` has only send and no receive/ack | The `destination`/`messageId` of send/receive are inconsistent | Confirm the receive's `destination.name`==exchange (not queue) |
| `events` has data but the trace cannot be linked | leader/partner did not inject an emitter, or the AMQP `traceparent` was not propagated | Confirm `create_group_manager` injects `message_emitter`; the partner `GroupPartnerMqClient` injects the emitter |
| `receiveCount` ≈ N×`sendCount` (N = number of group members) | **Correct**: FANOUT delivers to all partner queues | No action needed |
| `receiveCount` doubles abnormally | Own sends are not filtered, or someone else's task-result is mistakenly emitted as receive | Confirm the O-M2 filtering rules |
| Query endpoints return 503 | ClickHouse unreachable / Compactor watermark lagging | `curl :8123/ping`; confirm Writer + Compactor are running |
| `lifecycles`/`deadletters`/`throughput` return 404 | Profile not enabled | Set `[message]` `reliability_enabled=true`/`destination_enabled=true` and restart |
| `destinations/query` always returns 503 `STATE_SNAPSHOT_UNAVAILABLE` | The demo defaults to `NullDestinationStateSource` | **Expected** |
| Fluent Bit is up but messages do not reach Kafka | macOS missing `Workers 1` / wrong path / missing kafka.4 | Confirm `kafka.4 worker #0 started`; verify the absolute path of `amp_message*.jsonl` |
| `monitor-server.message.writer.v1` STATE Dead / LAG does not decrease | Uvicorn `--reload` repeatedly restarts lifespan because of writes to `logs/` | `just dev restart`; or `[server] reload=false`; confirm the Justfile contains `--reload-exclude 'logs/*'` |
| No `amp_message*.jsonl` after a group `/submit` | mq-auth-server is not started or the Redis port is wrong | Start `mq-auth-server`; `REDIS_URL=redis://localhost:6379/0` (**do not use the old 16379**); `just dev restart` demo-leader/partner; `just dev doctor` probes Redis connectivity |
| `rpk topic consume ... -o end` produces no output for a long time | This command waits for **new** messages and blocks once the partitions are fully consumed | Use `rpk topic consume amp.message -p <N> -o 0 --num 3` instead |

## 6. Relationship to Automated Tests

| Layer | Entry Point | Description |
|------|------|------|
| Development-mode runbook | This document | Walk the full pipeline manually, step by step |
| Smoke test | `scripts/e2e_message_verify.py` | Direct Kafka connection; only infra+monitor needed |
| Full-pipeline demo | `scripts/e2e_message_demo.sh` | All services online; assertions after group submit |
| Automated E2E | `just test e2e -k message` | pytest black-box test, runnable in CI |

See `plans/message-emitter-and-local-e2e-design.md` and `plans/message-code-structure-design.md` for details.
