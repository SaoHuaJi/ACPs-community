**[English](dev-runbook-audit_en.md) | [中文](dev-runbook-audit.md)**

# Development-Mode Audit Joint Verification

**Prerequisite**: first complete service startup (monitor-server, demo-leader,
demo-partner, Fluent Bit) per [dev-runbook_en.md](./dev-runbook_en.md). This document covers only the
verification content of the Audit pipeline in development mode.

The **signature-verification step** in the pipeline has two modes, both covered in this document:

- **Mock mode** ([Chapters 2–3](#2-generate-audit-logs)): purely local, the public key comes from a local file, no ca-server required; suited to day-to-day development and CI.
- **CA joint-verification mode** ([Chapter 4](#4-advanced-ca-joint-verification-ca-mode)): verifies signatures against ca-server in real time using the certificate serial number; aimed at integration testing / staging / production.

For a first read, get the basic Mock-mode pipeline in Chapters 2–3 working first, then upgrade to CA mode per Chapter 4.

## 1. Pipeline Overview

### 1.1 Data Flow and Ports

```text
demo-leader / demo-partner
  └─ acps-sdk AuditEmitter
        │ write NDJSON audit log
        ▼
  logs/amp_audit.jsonl             (demo-leader, one file)
  logs/amp_audit_*.jsonl           (demo-partner, one file per partner)
        │
        │ Fluent Bit tail + JSON parse
        ▼
  Kafka  amp.audit  (localhost:19092, Redpanda)
        │
        │ monitor-server Kafka Consumer
        │ verify signature → build hash chain → idempotent write
        ▼
  PostgreSQL  agent_monitor.audit_records  (localhost:5432)
        │
        │ Query API
        ▼
  http://localhost:9009/acps-amp-v1/audit/...
```

Components and ports involved:

| Component | Port | Description |
|------|------|------|
| acps-infra PostgreSQL | 5432 | Shared database for all servers |
| acps-infra Redpanda (Kafka) | 19092 | Audit event stream (host access) |
| demo-leader API | 9031 | Leader business API (triggers audit logs) |
| demo-leader Web UI | 9030 | Leader static frontend |
| demo-partner | 9023+ | Partner service instances (trigger audit logs) |
| monitor-server Query API | 9009 | Audit record query API |
| ca-server | 9003 | Certificate public-key query service (**required only in CA mode**) |

### 1.2 The Two Signature-Verification Modes

Signature verification happens in monitor-server's Kafka Consumer and is decided by a single switch,
`[atr].mock_mode` in `config/development.toml`:

| Feature | Mock mode (`mock_mode = true`, default) | CA mode (`mock_mode = false`) |
|------|-------------------------------|-------------------------------|
| Public key source | `config/audit_keys.json` (local file) | ca-server HTTP API |
| `integrity.kid` | Arbitrary string; must have a matching entry in `audit_keys.json` | X.509 certificate serial number (uppercase hex) |
| `integrity.alg` | Determined by the Agent signer: `"EdDSA"` or `"RS256"` | Determined by the certificate key type |
| Dependent services | None (purely local) | ca-server online |
| Certificate revocation awareness | No | Yes (status check) |
| Applicable scenarios | Purely local development, CI unit tests | Integration testing, staging, production |

> In either mode, **audit records are always persisted normally**; the verification result only affects
> the two fields `signature_verified` and
> `verification_failure_type` — audit data is never discarded because verification failed.

## 2. Generate Audit Logs

### 2.1 Trigger via the demo-leader API

Call demo-leader's `/api/v1/submit` endpoint to trigger a complete business request. At the key points
of request handling (submission, intent analysis, task scheduling, etc.), Leader automatically writes
audit logs to `demo-leader/logs/amp_audit.jsonl` through the `acps-sdk AuditEmitter`.

> **Note**: `/api/v1/submit` internally calls the LLM API, so in a development environment without an
> LLM configured it returns `LLM_CALL_ERROR`, but audit logs may still be partially written (depending
> on Leader's specific implementation).
> If the LLM is unavailable, you can write test logs directly with the SDK (see "Fallback" below).

```bash
curl -s -X POST http://localhost:9031/api/v1/submit \
  -H "Content-Type: application/json" \
  -d '{
    "clientRequestId": "audit-test-001",
    "query": "Help me look up high-speed trains from Beijing to Shanghai",
    "mode": "direct_rpc"
  }' | python3 -m json.tool
```

### 2.2 Fallback: Inline SDK Write (No LLM Environment)

**Option A — Ed25519 signing (Mock mode, recommended for day-to-day development)**

```python
cd demo-leader
uv run python - <<'EOF'
import sys
sys.path.insert(0, '../acps-sdk')
from pathlib import Path
from acps_sdk.amp.emitter import AuditEmitter
from acps_sdk.amp.signer import load_signer_from_keys_json
from acps_sdk.amp.models import AuditBody, AuditActor, AuditAction, AuditTarget, AuditResult

aic = '1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ'
signer = load_signer_from_keys_json('../monitor-server/config/audit_keys.json', aic)
print(f'kid={signer.kid}, alg={signer.alg}')  # audit-key-demo_leader / EdDSA

emitter = AuditEmitter(Path('logs/amp_audit.jsonl'), aic=aic, signer=signer)
body = AuditBody(
    actor=AuditActor(id='user-e2e-test', type='user', name='E2E Test'),
    action=AuditAction(name='e2e.verify', type='test'),
    target=AuditTarget(type='audit_log', id='e2e-target-001'),
    result=AuditResult(status='success'),
)
print('emitted log_id:', emitter.emit_sync(body))
EOF
```

**Option B — Certificate signing (CA / Mock mode, using client.pem)**

```python
cd demo-leader
uv run python - <<'EOF'
import sys
sys.path.insert(0, '../acps-sdk')
from pathlib import Path
from acps_sdk.amp.emitter import AuditEmitter
from acps_sdk.amp.signer import CertificateAuditSigner
from acps_sdk.amp.models import AuditBody, AuditActor, AuditAction, AuditTarget, AuditResult

signer = CertificateAuditSigner(
    private_key_pem=Path('leader/atr/client.key').read_text(),
    cert_pem=Path('leader/atr/client.pem').read_text(),
)
print(f'kid={signer.kid}, alg={signer.alg}')  # <cert serial> / RS256

emitter = AuditEmitter(Path('logs/amp_audit.jsonl'),
    aic='1.2.156.3088.1.1.SC64YN.Z5LSGY.1.0NMQ', signer=signer)
body = AuditBody(
    actor=AuditActor(id='user-e2e-test', type='user', name='E2E Test'),
    action=AuditAction(name='e2e.verify', type='test'),
    target=AuditTarget(type='audit_log', id='e2e-target-001'),
    result=AuditResult(status='success'),
)
print('emitted log_id:', emitter.emit_sync(body))
EOF
```

### 2.3 Inspect the Written Log Files

```bash
tail -1 demo-leader/logs/amp_audit.jsonl | python3 -m json.tool

if ls demo-partner/logs/amp_audit_*.jsonl >/dev/null 2>&1; then
  for f in demo-partner/logs/amp_audit_*.jsonl; do
    echo "== ${f} =="
    tail -1 "${f}" | python3 -m json.tool
  done
fi
```

## 3. Verify Each Stage

### 3.1 Verify Kafka (Fluent Bit has forwarded)

```bash
docker exec dev-redpanda rpk topic describe amp.audit -p
# Expected: HIGH-WATERMARK should increase after each business trigger

HW=$(docker exec dev-redpanda rpk topic describe amp.audit -p | awk '$1=="0"{print $6}')
OFFSET=$((HW - 1))
docker exec dev-redpanda rpk topic consume amp.audit \
  --partitions=0 --offset="${OFFSET}" --num=1 \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(json.loads(d['value']), indent=2, ensure_ascii=False))"
```

### 3.2 Verify PostgreSQL (monitor-server has persisted)

```bash
psql postgresql://monitor:monitor@localhost:5432/agent_monitor \
  -c "SELECT log_id, action_name, signature_verified, committed_at
      FROM audit_records
      ORDER BY committed_at DESC LIMIT 5;"
```

Expected: you can see the record corresponding to the Kafka message from the previous step, with `committed_at` being the most recent time.

> In Mock mode, if the corresponding public key is not injected into `MockKeyResolver`, `signature_verified` may be `false`
> and `verification_failure_type = missing_public_key`; this is expected behavior (see [§5 Troubleshooting](#5-troubleshooting)).

### 3.3 Verify the Query API

```bash
# Query all audit records within the last 10 minutes (macOS date syntax)
START=$(date -u -v-10M +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
curl -s -X POST http://localhost:9009/acps-amp-v1/audit/records/query \
  -H "Content-Type: application/json" \
  -d "{
    \"timeRange\": {\"startAt\": \"$START\", \"endAt\": \"$END\"},
    \"page\": {\"limit\": 10}
  }" | python3 -m json.tool
```

Expected: the `items` array contains the audit record just triggered, where:

- `integrity.signatureVerified` is `true` (in Mock mode the corresponding public key must exist in `audit_keys.json`)
- `integrity.signatureAlg`: `"EdDSA"` for Ed25519 keys; `"RS256"` for X.509 RSA certificates

> **About `dataFreshnessAt`**: the response's `meta.dataFreshnessAt` indicates the earliest event time
> that monitor-server has stably consumed and persisted (the minimum of the watermarks across all Kafka
> partitions). If a partition has had no new data for a long time, this value may look old; that is
> normal behavior and does not mean the latest data has not been persisted.

You can also test interactively through the OpenAPI docs: `http://localhost:9009/docs`

### 3.4 One-Shot End-to-End Script

After clearing the local audit logs, `scripts/demo_audit.sh` sends `/api/v1/submit` to demo-leader,
waits for Fluent Bit + AuditWriter propagation, then asserts via `traceId` that the monitor Query API
record count is ≥ 2.

```bash
# Any current directory works (the script resolves the acps/ root automatically); see dev-runbook.md §1.2
bash monitor-server/scripts/demo_audit.sh
```

## 4. Advanced: CA Joint Verification (CA Mode)

Building on Chapters 2–3, this chapter switches verification from the local Mock public key to
**querying ca-server in real time for the public key by certificate serial number (`kid`)**,
delivering a production-grade verification pipeline.

### 4.1 Design and Data Flow

```text
Agent's X.509 certificate (issued by acps-cli bootstrap)
  │ serial number → kid (uppercase hex)
  │ private key signs AMP logs
  ▼
audit.jsonl  integrity.kid = certificate serial number, integrity.alg = EdDSA / RS256
  │ Fluent Bit → Kafka
  ▼
monitor-server CAKeyResolver
  │ GET /acps-atr-v2/ca/keys/{kid}
  ▼
ca-server (certificate database inside PostgreSQL)
  │ returns publicKey PEM + status(valid/revoked/expired)
  ▼
monitor-server completes JCS + signature verification
  ▼
audit_records.signature_verified = true
```

### 4.2 Certificate Pre-Checks and ca-server Registration

#### 4.2.1 Check the Agent Certificate

```bash
# demo-leader
ls -la demo-leader/leader/atr/client.pem demo-leader/leader/atr/client.key
openssl x509 -in demo-leader/leader/atr/client.pem -serial -noout
# serial=5FCB77CA23BBA4A402358721986C16C7626269B
openssl x509 -in demo-leader/leader/atr/client.pem -text -noout | grep "Public Key Algorithm"
# rsaEncryption → alg = "RS256"

# demo-partner
ls demo-partner/partners/online/china_hotel/client.pem
openssl x509 -in demo-partner/partners/online/china_hotel/client.pem -serial -noout
```

#### 4.2.2 Verify ca-server Has Registered the Certificate

```bash
SERIAL=$(openssl x509 -in demo-leader/leader/atr/client.pem -serial -noout | cut -d= -f2 | sed 's/^0*//')
curl -s "http://localhost:9003/acps-atr-v2/ca/keys/${SERIAL}" | python3 -m json.tool
# Expected: status = "valid", publicKey non-empty
```

If it returns 404 (`CERTIFICATE_NOT_FOUND`), re-bootstrapping is required (see §4.2.3).

#### 4.2.3 When the Certificate Is Not in ca-server: Re-bootstrap

Re-issue the certificate through the ACME flow (this writes a new key pair and certificate; `integrity.kid` becomes the new certificate serial number):

```bash
curl -s http://localhost:9001/health  # confirm registry-server is online

cd acps-cli
./scripts/bootstrap.sh demo-leader --install-dir ../demo-leader
./scripts/bootstrap.sh demo-partner --install-dir ../demo-partner
```

After issuance completes, query ca-server once more using §4.2.2 to confirm `status = "valid"`.

### 4.3 Switch the Configuration to CA Mode

Edit `monitor-server/config/development.toml`:

```toml
[atr]
mock_mode = false                      # false = query ca-server for the public key in real time
ca_base_url = "http://localhost:9003"  # ca-server address
```

| Config item | Key | Default | Description |
|--------|-----|--------|------|
| CA service address | `[atr].ca_base_url` | `http://localhost:9003` | ca-server root address |
| Mock mode | `[atr].mock_mode` | `true` | When `false`, real CA queries are used |

#### Agent Side (no extra configuration needed)

`demo-leader` and `demo-partner` **automatically** prefer the CA certificate mode:

- If `leader/atr/client.pem` + `leader/atr/client.key` exist → `CertificateAuditSigner`
- Otherwise → `load_signer_from_keys_json` (mock-mode fallback)

### 4.4 Incrementally Start ca-server

On top of the services in [dev-runbook_en.md](./dev-runbook_en.md), additionally start ca-server and use the CA-mode configuration:

```bash
# Start ca-server
(cd ca-server && just dev start)
curl http://localhost:9003/health  # Expected: {"status":"healthy"}

# Confirm the Agent certificate is in ca-server (see §4.2.2)

# Configure monitor-server to use CA mode (see §4.3), then restart
(cd monitor-server && just dev restart)
```

### 4.5 CA-Mode Verification

#### 4.5.1 Agent Side: Confirm Signing Uses the Certificate Serial Number

```bash
tail -1 demo-leader/logs/amp_audit.jsonl | python3 -c "
import json, sys
r = json.load(sys.stdin)
print('kid:', r.get('integrity', {}).get('kid'))   # certificate serial number (uppercase hex)
print('alg:', r.get('integrity', {}).get('alg'))   # RS256 or EdDSA
print('sig:', (r.get('integrity', {}).get('sig') or '')[:20] + '...')
"
```

#### 4.5.2 ca-server Returns the Corresponding Public Key

```bash
KID=$(tail -1 demo-leader/logs/amp_audit.jsonl | python3 -c "import json,sys; print(json.load(sys.stdin)['integrity']['kid'])")
curl -s "http://localhost:9003/acps-atr-v2/ca/keys/${KID}" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('status:', d.get('status'))     # should be valid
print('publicKey (first 60 chars):', d.get('publicKey','')[:60])
"
```

#### 4.5.3 signature_verified = true in PostgreSQL

```bash
psql postgresql://monitor:monitor@localhost:5432/agent_monitor \
  -c "SELECT log_id, signature_verified, verification_failure_type, committed_at
      FROM audit_records
      ORDER BY committed_at DESC LIMIT 5;"
# Expected: signature_verified = t, verification_failure_type = NULL
```

#### 4.5.4 Quick E2E Verification Script (does not depend on Kafka/Fluent-Bit)

```bash
cd monitor-server
APP_ENV=development uv run python scripts/smoke_audit.py
# [PASS] CA-based audit log signing + verification E2E passed!
```

### 4.6 Certificate Revocation Behavior

After a certificate is revoked by ca-server (`status = revoked` or `expired`):

1. `CAKeyResolver._fetch_from_ca()` checks the `status` field and throws `KeyNotFoundError` if it is not `valid`.
2. `AuditWriter` catches `KeyNotFoundError` and sets `verification_failure_type = "missing_public_key"`.
3. The record is still persisted normally (audit data is never discarded), but `signature_verified = false`.

## 5. Troubleshooting

### 5.1 No Messages in Kafka

See the general troubleshooting in [dev-runbook_en.md §5](./dev-runbook_en.md).

Confirm the Audit topic exists and its watermark is growing:

```bash
docker exec dev-redpanda rpk topic describe amp.audit -p
```

Confirm the consumer group lag → 0:

```bash
docker exec dev-redpanda rpk group describe amp.audit.writer
```

If demo-partner has no log files, confirm the instance is running:

```bash
just -f demo-partner/Justfile app status
ls demo-partner/logs/
```

### 5.2 Mock Mode: signature_verified = false (missing_public_key)

**Signing-mode notes**:

- If demo-leader / demo-partner have `client.pem` + `client.key`, they automatically use
  `CertificateAuditSigner` (`kid` = certificate serial number, `alg` = `"RS256"` or `"EdDSA"`).
- Otherwise they fall back to `load_signer_from_keys_json()` (`alg = "EdDSA"`, `kid` = a hand-written string).

**Ed25519 signing (fallback mode)**: run the script to generate/update `audit_keys.json`:

```bash
cd monitor-server
uv run python scripts/gen_audit_keys.py
```

**RSA certificate signing (CertificateAuditSigner)**: manually add the certificate public key to `config/audit_keys.json`:

```bash
# Get the serial number and SPKI public key
openssl x509 -in demo-leader/leader/atr/client.pem -serial -noout
openssl x509 -in demo-leader/leader/atr/client.pem -pubkey -noout
```

```json
{
  "demo_leader_cert": {
    "aic": "<demo-leader's AIC>",
    "kid": "<certificate serial number, uppercase hex, leading zeros removed>",
    "public_key": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----\n",
    "private_key": ""
  }
}
```

### 5.3 CA Mode: Troubleshooting Points

**Confirm the configuration has taken effect**:

```bash
cd monitor-server && APP_ENV=development uv run python -c "
from app.core.config import settings
print('mock_mode:', settings.atr_mock_mode)
print('ca_base_url:', settings.atr_ca_base_url)
"
```

**Common causes**:

1. `mock_mode` is still `true` (it uses MockKeyResolver and does not query ca-server).
2. ca-server is not running: `curl http://localhost:9003/health`.
3. The certificate is not in ca-server: `curl http://localhost:9003/acps-atr-v2/ca/keys/{KID}`.
   If it returns 404, perform the import steps in §4.2.3.
4. The Agent side is not using `CertificateAuditSigner` (the certificate files do not exist):
   ```bash
   ls demo-leader/leader/atr/client.pem demo-leader/leader/atr/client.key
   ```

**ca-server unreachable (ATRUnavailableError)**: monitor-server does not lose audit data when it cannot connect —
records are still persisted, with `signature_verified = false` and `verification_failure_type = "missing_public_key"`.
After ca-server recovers, newly arriving messages are automatically re-verified (but already-persisted records are not backfilled).

**No new records in audit_records**: confirm the consumer group has no backlog; if there is one, check the monitor-server logs:

```bash
just -f monitor-server/Justfile app logs
```
