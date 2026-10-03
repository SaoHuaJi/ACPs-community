## 5. CA User-Side Commands

### 5.1 Shared Group Options

The command groups below all inherit the root-level `--config` and `--verbose`, and additionally provide:

- `acps-cli cert --server-url`

Meaning: overrides the CA service base address.

### acps-cli cert issue

Purpose: request a new certificate for an Agent.

Parameters:

- `--aic, -a`: required, Agent Identity Code
- `--eab-file`: required, path to the EAB JSON file
- `--usage, -u`: required, certificate usage, one of `clientAuth` or `serverAuth`
- `--key-type, -k`: key type, one of `ec` or `rsa`, default `ec`
- `--reuse-key`: reuse the local private key if one already exists
- `--key-path`: output path for the private key
- `--cert-path`: output path for the certificate chain
- `--trust-bundle-path`: output path for the trust bundle

Description: the JSON pointed to by `--eab-file` must contain the `keyId` and `macKey` fields plus an `aic` field consistent with `--aic`. A successful issuance also updates the trust bundle.

### acps-cli cert renew

Purpose: renew an existing certificate.

Parameters:

- `--aic, -a`: required, Agent Identity Code
- `--eab-file`: required, path to the EAB JSON file
- `--usage, -u`: required, certificate usage, one of `clientAuth` or `serverAuth`
- `--force, -f`: force renewal even if the certificate is not yet close to expiry
- `--key-path`: output path for the private key
- `--cert-path`: output path for the certificate chain
- `--trust-bundle-path`: output path for the trust bundle

Description: without `--force`, the command refuses to renew if the local certificate still has more than 30 days of validity remaining. Renewal reuses the local Agent private key.

### acps-cli cert revoke

Purpose: revoke a certificate.

Parameters:

- `--aic, -a`: required, Agent Identity Code
- `--reason, -r`: revocation reason, default `unspecified`

Description: the current implementation recognizes `unspecified`, `keyCompromise`, `cACompromise`, `affiliationChanged`, `superseded` and `cessationOfOperation`; any other string is handled as `unspecified`.

### acps-cli cert status

Purpose: query the current certificate status of an Agent.

Parameters:

- `--aic, -a`: required, Agent Identity Code
- `--cert-path`: local certificate file path
- `--check-ocsp/--no-check-ocsp`: whether to perform an OCSP check, enabled by default, i.e. `--check-ocsp`

### acps-cli cert account-key rollover

Purpose: roll over the ACME account key.

Parameters:

- `--aic, -a`: required, Agent Identity Code
- `--new-key, -n`: path to the new key file; may point to a pre-generated key, or serve as the output path for automatic generation
- `--key-type, -k`: key type used for automatic generation, one of `ec` or `rsa`, default `ec`
- `--backup/--no-backup`: whether to back up the old account key first, enabled by default, i.e. `--backup`

Description: if `--new-key` points to an existing file, the command reads that pre-generated key; if the path does not exist, it generates a new key and writes it to that path after a successful rollover. When `--new-key` is not provided, only the account key file of the current AIC is updated.

### acps-cli cert trust-bundle update

Purpose: update the local trust bundle file.

Parameters:

- `--output, -o`: output path

### acps-cli cert crl download

Purpose: download a CRL file.

Parameters:

- `--output, -o`: output file path
- `--format, -f`: CRL format, one of `der` or `pem`, default `der`
- `--version`: download a historical CRL version, only applicable to the DER download path

Description: when `--output` is not provided, the current DER CRL is saved by default to `ca.crl` in the CA certificate directory, PEM is saved to `ca.pem`, and historical versions are saved to `ca-<version>.crl`.

### acps-cli cert crl info

Purpose: view the current CRL metadata.

Parameters: no dedicated parameters.

### acps-cli cert crl detail

Purpose: view details of the revocation entries in the current CRL.

Parameters: no dedicated parameters.

### acps-cli cert ocsp check

Purpose: check certificate status via OCSP.

Parameters:

- `--aic, -a`: Agent Identity Code
- `--cert, -c`: certificate file path
- `--issuer, -i`: issuer certificate or trust bundle file path
- `--request-method`: OCSP request method, one of `post` or `get`, default `post`
- `--json`: output the result as JSON

Description: the common usage is to pass an AIC, or to pass a certificate file directly for the check.

### acps-cli cert ocsp cert-status

Purpose: call the simplified OCSP status query endpoint.

Parameters:

- `--serial-number`: certificate serial number
- `--aic`: Agent Identity Code
- `--cert-path`: certificate file path used to extract the serial number

Description: choose any one of the three input methods according to your actual environment.

## 6. Discovery User-Side Commands

### 6.1 Shared Group Options

`acps-cli discover` inherits the root-level `--config` and `--verbose`, and additionally provides:

- `--server-url`: overrides the Discovery service base address

### acps-cli discover status

Purpose: check the Discovery service status.

Parameters: no dedicated parameters.

### acps-cli discover query

Purpose: perform a Discovery query; supports both natural-language queries and a structured request body.

Parameters:

- `QUERY_STR`: optional positional argument, natural-language query text
- `--type`: explicitly specify the Discovery request type; when not provided and the request body has no `type`, defaults to `explicit`
- `--limit`: maximum number of returned records, range `1-50`; when not provided and the request body has no `limit`, defaults to `5`
- `--request-json`: inline DiscoveryRequest JSON; cannot be used together with `--request-file`
- `--request-file`: path to a DiscoveryRequest JSON file; cannot be used together with `--request-json`
- `--filter-json`: inline DiscoveryFilter JSON; cannot be used together with `--filter-file`
- `--filter-file`: path to a DiscoveryFilter JSON file; cannot be used together with `--filter-json`
- `--context-json`: inline query context JSON; cannot be used together with `--context-file`
- `--context-file`: path to a query context JSON file; cannot be used together with `--context-json`
- `--forward-depth-limit`: forwarding depth limit, range `1-5`
- `--forward-fanout-limit`: forwarding fan-out limit, range `1-5`
- `--forward-fanout-remaining`: remaining fan-out budget, range `0-5`
- `--forward-chain`: append a `forwardChain` entry, repeatable
- `--forward-trusted-server`: append a `forwardTrustedServers` entry, repeatable
- `--forward-signature`: append a `forwardSignatures` entry, repeatable
- `--forward-each-timeout-ms`: per-hop forwarding timeout (milliseconds), must be greater than or equal to `1`
- `--forward-total-timeout-ms`: total forwarding timeout (milliseconds), must be greater than or equal to `1`

Description:

- For simple queries, only `QUERY_STR` needs to be passed
- When precise control over the request structure is needed, prefer `--request-json` or `--request-file`
- When `--request-json` or `--request-file` is used, options such as `QUERY_STR`, `--limit` and the forward options on the command line override fields of the same name in the request body
- The Forward-related parameters are mainly used in multi-hop forwarding and chain testing scenarios

## 7. Monitor User-Side Commands

### 7.1 Shared Group Options

`acps-cli monitor` inherits the root-level `--config` and `--verbose`, and additionally provides:

- `--server-url`: overrides the monitor-server root address
- `--api-prefix`: overrides the AMP Query API prefix, default is `/acps-amp-v1`

The default configuration section is as follows:

```toml
[monitor]
base_url = "http://localhost:9009"
api_prefix = "/acps-amp-v1"
timeout_seconds = 15
```

Description:

- `monitor` configuration precedence follows: command-line options > `MONITOR_*` environment variables > `[monitor]` > default values
- `monitor status` accesses `GET /health` and does not append `api_prefix`
- The remaining business commands access `{base_url}{api_prefix}/...`
- `monitor` query commands output pretty JSON by default
- All POST query commands support `--request-json` or `--request-file`
- `--request-json` and `--request-file` are mutually exclusive, and the content must be a JSON object
- When a request and shortcut parameters are provided at the same time, the CLI uses the shortcut parameters to override scalar fields of the same name, `timeRange` and `page` in the request body
- Shortcut filters are only merged into a simple `filter` that has `logic=and` and no `groups`; for complex filtering, write the complete request directly
- Once a complete request is provided, the CLI no longer requires the local mandatory parameters that exist only to "build the payload via shortcuts"
- payload-only commands do not automatically construct an empty request body; the request must be provided explicitly
- This command group only handles Query API queries and troubleshooting; it does not include smoke, Kafka writes or AMP LogRecord construction; write-path verification still uses `monitor-server/scripts/smoke_*.py`

### acps-cli monitor status

Purpose: check the health status of monitor-server.

Parameters: no dedicated parameters.

### acps-cli monitor heartbeat

Purpose: query the Heartbeat read model.

Subcommands:

- `summary`: query the global Heartbeat summary
- `liveness <AIC>`: point-query the liveness of a single AIC
- `query`: batch-query liveness snapshots

`heartbeat query` parameters:

- `--aic`: append an `aic eq` filter
- `--limit`: write to `page.limit`
- `--cursor`: write to `page.cursor`
- `--request-json` / `--request-file`: submit a complete request body

Description:

- When no request is provided, `heartbeat query` must provide `--aic`
- Heartbeat liveness query does not support shortcut `--start` / `--end` time range construction

### acps-cli monitor metrics

Purpose: query the Metrics read model.

Subcommands:

- `snapshots`: query the latest metrics snapshots
- `series`: query metrics time series
- `rankings`: query metrics rankings

`metrics snapshots` parameters:

- `--aic`: append an `aic eq` filter
- `--limit` / `--cursor`: write to `page`
- `--request-json` / `--request-file`: submit a complete request body

`metrics series` parameters:

- `--metric`: write to the top-level `metric`
- `--aic`: append an `aic eq` filter
- `--start`, `--end`: write to `timeRange.startAt` / `timeRange.endAt`
- `--step`: write to the top-level `step`
- `--request-json` / `--request-file`: submit a complete request body

`metrics rankings` parameters:

- `--metric`: write to the top-level `metric`
- `--start`, `--end`: write to `timeRange`
- `--top-n`: write to the top-level `topN`
- `--request-json` / `--request-file`: submit a complete request body

Description:

- When no request is provided, both `metrics series` and `metrics rankings` must provide `--metric`, `--start` and `--end`
- If the Analytics Profile is not enabled on the server side, `rankings` passes through the `404/422/503` returned by the server directly

### acps-cli monitor access

Purpose: query Access events, Trace and analytics read models.

Subcommands:

- `events`
- `operations`
- `traces`
- `trace <TRACE_ID>`
- `slow`
- `errors`
- `topology`

`access events` parameters:

- `--aic`: append an `aic eq` filter
- `--trace-id`: append a `traceId eq` filter
- `--start`, `--end`: write to `timeRange`
- `--limit` / `--cursor`: write to `page`
- `--request-json` / `--request-file`: submit a complete request body

`access trace <TRACE_ID>` parameters:

- `--include-events`: mapped to the GET query parameter `include_events=true`

payload-only subcommands:

- `operations`
- `traces`
- `slow`
- `errors`
- `topology`

Description:

- When no request is provided, `access events` must provide `--start` and `--end`
- The payload-only subcommands listed above must explicitly provide `--request-json` or `--request-file`

### acps-cli monitor message

Purpose: query Message events, lifecycles, dead letters and destination read models.

Subcommands:

- `events`
- `lifecycles`
- `lifecycle <MESSAGE_ID>`
- `deadletters`
- `destinations`
- `throughput`

`message events` parameters:

- `--message-id`: append a `messageId eq` filter
- `--trace-id`: append a `traceId eq` filter
- `--start`, `--end`: write to `timeRange`
- `--limit` / `--cursor`: write to `page`
- `--request-json` / `--request-file`: submit a complete request body

`message lifecycle <MESSAGE_ID>` parameters:

- `--system`: mapped to the query parameter `system`
- `--destination-name`: mapped to the query parameter `destinationName`
- `--destination-kind`: mapped to the query parameter `destinationKind`
- `--virtual-host`: mapped to the query parameter `virtualHost`

payload-only subcommands:

- `lifecycles`
- `deadletters`
- `destinations`
- `throughput`

Description:

- When no request is provided, `message events` must provide `--start` and `--end`
- The payload-only subcommands listed above must explicitly provide `--request-json` or `--request-file`
- Whether Reliability / Destination Profile is enabled is decided by the server side; the CLI only handles requests and error pass-through

### acps-cli monitor system

Purpose: query System log events.

Subcommands:

- `events`

`system events` parameters:

- `--aic`: append an `aic eq` filter
- `--correlation-id`: append a `correlationId eq` filter
- `--severity-min`: append a `severityNumber gte` filter
- `--start`, `--end`: write to `timeRange`
- `--limit` / `--cursor`: write to `page`
- `--request-json` / `--request-file`: submit a complete request body

Description:

- When no request is provided, `system events` must provide `--start` and `--end`

### acps-cli monitor audit

Purpose: query Audit records, chain anchors and integrity verification tasks.

Subcommands:

- `records`
- `record <AUDIT_ID>`
- `anchors`
- `verify`
- `verify-task <TASK_ID>`

`audit records` parameters:

- `--aic`: append an `aic eq` filter
- `--keyword`: write to the top-level `keyword`
- `--start`, `--end`: write to `timeRange`
- `--limit` / `--cursor`: write to `page`
- `--request-json` / `--request-file`: submit a complete request body

`audit anchors` parameters:

- `--chain-id`: mapped to the GET query parameter `chain_id`

`audit verify` parameters:

- `--request-json` / `--request-file`: submit a complete verification request body

Description:

- When no request is provided, `audit records` must provide `--start` and `--end`
- `audit verify` is a payload-only command and must explicitly provide a request
- `audit verify` only submits a verification task to the server; it does not perform signing, public-key parsing or hash-chain computation locally
