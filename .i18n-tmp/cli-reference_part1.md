[Home](../README.md)

**[English](cli-reference_en.md) | [中文](cli-reference.md)**

# acps-cli Command-Line Reference

This document is the complete command-line reference for `acps-cli`.

## 1. Usage Conventions

### 1.1 How to Run It

- Common form in a development environment: `uv run acps-cli ...`
- Once installed into a virtual environment or the system environment, run it directly: `acps-cli ...`

### 1.2 Configuration File Load Order

The root command supports two common options:

- `--config PATH`: explicitly specify the configuration file path
- `--verbose`: enable more verbose CLI log output

The CLI searches for the configuration file in the following order:

1. The file explicitly specified by `--config PATH`
2. `acps-cli.toml` in the current directory
3. `~/.acps-cli.toml` in the user directory

After finding a configuration file, the CLI additionally loads the `.env` in the same directory as that configuration file, and does not override environment variables already present in the current shell. If no configuration file is found at all, the CLI falls back to `python-dotenv`'s default `.env` lookup logic and returns an empty TOML configuration; commands that depend on service addresses, certificate paths, or token files usually still need those supplied explicitly through arguments or the environment.

The general precedence of configuration values is: command-line options > environment variables or `.env` > TOML configuration > code defaults. The unified entry point rejects some legacy configuration names and legacy environment variables, for example `[registry].server_base_url`, `REGISTRY_SERVER_BASE_URL`, `[ca].server_base_url`, `CA_SERVER_BASE_URL`, `[discovery].server_base_url`, `DISCOVERY_SERVER_BASE_URL`; use the new names listed below.

### 1.3 Output and Argument Style

- Most API-facing commands support `--json`, which outputs machine-readable JSON.
- Discovery queries and most CA administrative query commands already output JSON by themselves; such commands do not necessarily offer `--json` in addition.
- Registry-related commands generally support `--server-url` to temporarily override the primary service address; `acps-cli entity` additionally supports `--mtls-url` to override the `9002` mTLS plane address.
- CA- and Discovery-related commands support `--server-url` to temporarily override the service address.
- MQ administrative commands provide `--group-api-url` and `--auth-api-url` on the `acps-cli admin mq` group to temporarily override the two service addresses; some subcommands still support `--cert-file` and `--key-file` to override certificate material.
- Repeatable parameters are marked as "repeatable" below. Such parameters can be passed multiple times.
- Paired boolean switches are written as `--foo/--no-foo`; the default state is noted below.

### 1.4 Option Position

Group-level options (for example `--config`, `--verbose`, `--server-url`, `--mtls-url`) may be written either before or after the corresponding subcommand; the CLI resolves them according to the command that owns the option and does not require a fixed order. The following two forms are equivalent:

```text
acps-cli entity --mtls-url https://registry.example:8443 derive --ontology-aic <AIC>
acps-cli entity derive --mtls-url https://registry.example:8443 --ontology-aic <AIC>
```

When the same option name is declared by commands at several levels at once, an occurrence written before a subcommand belongs to the current group, and one written after that subcommand belongs to the deeper command. For example, `cert --server-url` overrides the CA address, while `cert eab --server-url` overrides the Registry address:

```text
acps-cli cert --server-url https://ca.example eab fetch --server-url https://registry.example --aic <AIC> --output eab.json
```

When viewing help, put `--help` after the command level you want to inspect: `acps-cli entity --help` shows the `entity` group options, and `acps-cli entity derive --help` shows the `derive` options.

## 2. Configuration Sections at a Glance

The default configuration sample is located at `acps-cli.toml` in the project root and is divided into the following main configuration sections:

- `[registry]`: Registry base address, `9002` mTLS plane address, ontology certificate directory, server CA file, request timeout, etc.
- `[auth]`: paths to the regular-user and administrator token files
- `[ca]`: CA service address, account key directory, private key directory, certificate directory, CSR directory, trust bundle path
- `[discovery]`: Discovery service address
- `[monitor]`: monitor-server address, AMP Query API prefix, HTTP timeout
- `[mq]`: mq-auth-server Group API/Auth API addresses, Leader certificate, probe certificate, server CA file, timeout

It is recommended to check the following before you start:

- `[registry].base_url`
- `[registry].mtls_base_url`
- `[ca].base_url`
- `[discovery].base_url`
- `[monitor].base_url`
- `[monitor].api_prefix`
- `[monitor].timeout_seconds`
- `[mq].group_api_url`
- `[mq].auth_api_url`

### 2.1 Common Environment Variable Overrides

The following environment variables override the configuration of the same name in the TOML file; command-line options still take the highest precedence:

| Domain | Environment variable |
| -- | -------- |
| Registry | `REGISTRY_BASE_URL`, `REGISTRY_MTLS_BASE_URL`, `REGISTRY_TIMEOUT_SECONDS`, `REGISTRY_ONTOLOGY_MTLS_MATERIALS_DIR`, `REGISTRY_MTLS_SERVER_CA_FILE` |
| Registry authentication | `AUTH_USER_TOKEN_FILE`, `AUTH_ADMIN_TOKEN_FILE`, `REGISTRY_USER_USERNAME`, `REGISTRY_USER_PASSWORD`, `REGISTRY_USER_NAME`, `REGISTRY_USER_ORG_NAME`, `REGISTRY_ADMIN_USERNAME`, `REGISTRY_ADMIN_PASSWORD` |
| CA | `CA_BASE_URL`, `CA_SERVER_ADMIN_API_TOKEN`, `CA_ACCOUNT_KEYS_DIR`, `CA_PRIVATE_KEYS_DIR`, `CA_CERTS_DIR`, `CA_CSR_DIR`, `CA_TRUST_BUNDLE_PATH` |
| Discovery | `DISCOVERY_BASE_URL` |
| Monitor | `MONITOR_BASE_URL`, `MONITOR_API_PREFIX`, `MONITOR_TIMEOUT_SECONDS` |
| MQ | `MQ_GROUP_API_URL`, `MQ_AUTH_API_URL`, `MQ_GROUP_CERT_FILE`, `MQ_GROUP_KEY_FILE`, `MQ_PROBE_CERT_FILE`, `MQ_PROBE_KEY_FILE`, `MQ_CA_FILE`, `MQ_TIMEOUT_SECONDS` |

## 3. Complete Command Tree

```text
acps-cli
├── auth
│   ├── login
│   ├── logout
│   ├── change-password
│   └── whoami
├── agent
│   ├── list
│   ├── save
│   ├── submit
│   ├── check
│   ├── sync
│   └── delete
├── entity
│   └── derive
├── cert
│   ├── eab
│   │   └── fetch
│   ├── issue
│   ├── renew
│   ├── revoke
│   ├── status
│   ├── account-key
│   │   └── rollover
│   ├── trust-bundle
│   │   └── update
│   ├── crl
│   │   ├── download
│   │   ├── info
│   │   └── detail
│   └── ocsp
│       ├── check
│       └── cert-status
├── discover
│   ├── status
│   └── query
├── monitor
│   ├── status
│   ├── heartbeat
│   │   ├── summary
│   │   ├── liveness
│   │   └── query
│   ├── metrics
│   │   ├── snapshots
│   │   ├── series
│   │   └── rankings
│   ├── access
│   │   ├── events
│   │   ├── operations
│   │   ├── traces
│   │   ├── trace
│   │   ├── slow
│   │   ├── errors
│   │   └── topology
│   ├── message
│   │   ├── events
│   │   ├── lifecycles
│   │   ├── lifecycle
│   │   ├── deadletters
│   │   ├── destinations
│   │   └── throughput
│   ├── system
│   │   └── events
│   └── audit
│       ├── records
│       ├── record
│       ├── anchors
│       ├── verify
│       └── verify-task
└── admin
    ├── auth
    │   ├── login
    │   ├── logout
    │   ├── change-password
    │   └── whoami
    ├── registry
    │   ├── review
    │   │   ├── list
    │   │   ├── approve
    │   │   └── reject
    │   ├── agent
    │       ├── disable
    │       └── enable
    │   └── user
    │       └── reset-password
    ├── ca
    │   ├── crl
    │   │   ├── list
    │   │   └── refresh
    │   └── ocsp
    │       ├── responder-info
    │       └── stats
    ├── discovery
    │   ├── run-sync
    │   └── dsp
    │       ├── status
    │       ├── registry-info
    │       ├── sync
    │       ├── start
    │       ├── stop
    │       ├── reset
    │       ├── hard-reset
    │       └── register-webhook
    └── mq
        ├── health
        ├── group
        │   ├── add-member
        │   ├── remove-member
        │   ├── delete
        │   └── kick
        └── auth-probe
            ├── user
            ├── vhost
            ├── resource
            └── topic
```

## 4. Registry User-Side Commands

### 4.1 Shared Group Options

The following command groups all inherit the root-level `--config` and `--verbose`, and additionally provide Registry group options:

- `acps-cli auth`
- `acps-cli agent`
- `acps-cli entity`
- `acps-cli cert eab`

The Registry shared group options are:

- `--server-url`: override the Registry service base address

Among them, `acps-cli entity` additionally supports:

- `--mtls-url`: override the Registry `9002` mTLS service address

### acps-cli auth login

Purpose: user login; if the account does not exist locally, the underlying logic can automatically register it using the supplied parameters.

Arguments:

- `--username`: username; if not provided, it usually falls back to interactive input
- `--password`: password; if not provided, it usually falls back to interactive input
- `--name`: display name used during automatic registration
- `--org-name`: organization name used during automatic registration
- `--json`: output the result as JSON

### acps-cli auth whoami

Purpose: view the current Registry user identity.

Arguments:

- `--json`: output the result as JSON

### acps-cli auth logout

Purpose: log out the current Registry user and clear the locally stored user token.

Arguments:

- `--json`: output the result as JSON

### acps-cli auth change-password

Purpose: interactively change the password of the currently logged-in user.

Arguments:

- `--json`: output the result as JSON

Description: the command prompts in turn for the current password and the new password, and requires the new password to be confirmed a second time. On success it additionally prints "Password changed; logging in again is recommended".

### acps-cli agent list

Purpose: list the Agent drafts or submitted records under the current user.

Arguments:

- `--page`: page number, default `1`
- `--page-size`: number of entries per page, default `20`
- `--status`: filter by status; can be passed repeatedly
- `--json`: output the result as JSON

### acps-cli agent save

Purpose: create or update an Agent draft.

Arguments:

- `--logo-url`: Agent logo URL
- `--acs-file`: required, path to the ACS JSON file
- `--ontology/--no-ontology`: whether to mark this Agent as ontology, default `--no-ontology`
- `--json`: output the result as JSON

### acps-cli agent submit

Purpose: submit an Agent draft into the review process.

Arguments:

- `--agent-id`: required, UUID of the Agent draft to submit
- `--json`: output the result as JSON

### acps-cli agent check

Purpose: check the review status of the corresponding Agent based on the local ACS file.

Arguments:

- `--acs-file`: required, path to the ACS JSON file
- `--json`: output the result as JSON

### acps-cli agent sync

Purpose: sync the latest ACS state from the server back to the local file.

Arguments:

- `--acs-file`: required, path to the ACS JSON file
- `--json`: output the result as JSON

### acps-cli agent delete

Purpose: delete an Agent draft owned by the current user.

Arguments:

- `--acs-file`: required, path to the ACS JSON file
- `--json`: output the result as JSON

### acps-cli entity derive

Purpose: derive and register an entity based on an ontology AIC that has already been approved.

Arguments:

- `--mtls-url`: override the Registry `9002` mTLS service address. This option is defined on the `entity` group and may be written before or after `derive`
- `--ontology-aic`: required, an approved ontology AIC
- `--payload-file`: optional, path to the UTF-8 JSON file containing the derived entity payload
- `--mtls-cert-file`: override the ontology mTLS certificate path
- `--mtls-key-file`: override the ontology mTLS private key path
- `--mtls-server-ca-file`: override the CA file used to verify the Registry `9002` server certificate
- `--json`: output the result as JSON

Description: this command depends on the Registry mTLS plane and usually requires `[registry].mtls_base_url` and the corresponding certificate material to be configured correctly.

`--payload-file` may be omitted. If provided, it must be a JSON object, and the allowed fields are:

- `endPoints`: the entity's own list of service endpoints. If omitted, the ontology endpoints are used
- `entityUserId`: bind an end-user ID
- `entityMeta`: custom entity metadata
- `certificate`: entity certificate issuance configuration; it is written into the entity ACS so that the SANs and the expected validity period can be read later when applying for a server certificate. It does not inherit the ontology's `certificate`

`ontologyAic` is not written in the file; it is passed via `--ontology-aic`. Other fields are rejected.

Example:

```json
{
  "entityUserId": "user-001",
  "entityMeta": { "scenario": "production" },
  "endPoints": [
    {
      "url": "https://entity.example.com/callback",
      "transport": "JSONRPC",
      "security": []
    }
  ],
  "certificate": {
    "altNames": {
      "dns": ["entity.example.com"],
      "ip": ["10.0.1.50"]
    },
    "requestedValidity": 365
  }
}
```

### acps-cli cert eab fetch

Purpose: obtain EAB credentials for a subsequent certificate request.

Arguments:

- `--aic`: required, the target Agent AIC
- `--output`: required, the EAB JSON output path
- `--json`: output the result as JSON

Description: although the command path sits under `cert eab`, it actually accesses a Registry capability.

Note: the `cert` group's own `--server-url` overrides the CA service address; EAB uses the Registry address, so the Registry override must be written after `cert eab`, for example `acps-cli cert eab --server-url http://localhost:9001 fetch ...`.

