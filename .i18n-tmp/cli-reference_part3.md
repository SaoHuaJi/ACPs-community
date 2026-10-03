## 8. Registry admin-side commands

### 8.1 Shared group options

The following admin command groups inherit the root-level `--config` and `--verbose`, and additionally provide:

- `acps-cli admin auth`
- `acps-cli admin registry`

Shared group options:

- `--server-url`: overrides the Registry service base address

### acps-cli admin auth login

Purpose: Registry administrator login.

Parameters:

- `--username`: administrator username
- `--password`: administrator password
- `--json`: output the result as JSON

### acps-cli admin auth whoami

Purpose: show the current Registry administrator identity.

Parameters:

- `--json`: output the result as JSON

### acps-cli admin auth logout

Purpose: log out the current Registry administrator and clear the locally saved administrator token.

Parameters:

- `--json`: output the result as JSON

### acps-cli admin auth change-password

Purpose: interactively change the password of the currently logged-in administrator.

Parameters:

- `--json`: output the result as JSON

Description: the command prompts for the current password and the new password in turn, and requires the new password to be confirmed again. On success it additionally prints "password changed, re-login recommended".

### acps-cli admin registry review list

Purpose: list Agent review requests that are pending review or in a specified status.

Parameters:

- `--page`: page number, default `1`
- `--page-size`: number of entries per page, default `20`
- `--status`: filter by status, repeatable
- `--json`: output the result as JSON

### acps-cli admin registry review approve

Purpose: approve the review of the specified Agent.

Parameters:

- `--agent-id`: required, Agent UUID
- `--comments`: optional review comments
- `--json`: output the result as JSON

### acps-cli admin registry review reject

Purpose: reject the review of the specified Agent.

Parameters:

- `--agent-id`: required, Agent UUID
- `--comments`: required, reason for rejection
- `--json`: output the result as JSON

### acps-cli admin registry agent disable

Purpose: disable an existing Agent.

Parameters:

- `--agent-id`: required, Agent UUID
- `--reason`: reason for disabling, default `Staff disable`
- `--json`: output the result as JSON

### acps-cli admin registry agent enable

Purpose: re-enable a disabled Agent.

Parameters:

- `--agent-id`: required, Agent UUID
- `--json`: output the result as JSON

### acps-cli admin registry user reset-password

Purpose: an administrator resets the password for the specified user.

Parameters:

- `--user-id`: required, target user UUID
- `--new-password`: optional, the target user's new password; if not provided, an interactive prompt with a second confirmation is used
- `--json`: output the result as JSON

## 9. CA admin-side commands

### 9.1 Shared group options

`acps-cli admin ca` inherits the root-level `--config` and `--verbose`, and additionally provides:

- `--server-url`: overrides the CA service base address

### acps-cli admin ca crl list

Purpose: list the CRL history recorded on the CA side.

Parameters:

- `--status`: filter by status, values `current`, `superseded`, `expired`
- `--page`: page number, default `1`
- `--page-size`: number of entries per page, default `20`

### acps-cli admin ca crl refresh

Purpose: refresh the current CRL.

Parameters: no dedicated parameters.

### acps-cli admin ca ocsp responder-info

Purpose: show OCSP responder metadata.

Parameters: no dedicated parameters.

### acps-cli admin ca ocsp stats

Purpose: show OCSP service statistics.

Parameters: no dedicated parameters.

## 10. Discovery admin-side commands

### 10.1 Shared group options

`acps-cli admin discovery` inherits the root-level `--config` and `--verbose`, and additionally provides:

- `--server-url`: overrides the Discovery service base address

### acps-cli admin discovery run-sync

Purpose: trigger one Discovery orchestration-level sync.

Parameters:

- `--hard-reset/--no-hard-reset`: whether to clear Discovery data before syncing, enabled by default, i.e. `--hard-reset`
- `--expect-acs-min`: the minimum ACS count required after the sync completes, default `1`
- `--skip-acs-check`: skip the ACS count validation

### acps-cli admin discovery dsp status

Purpose: show the current DSP status.

Parameters:

- `--expect-acs-min`: require the number of ACS objects in the returned result to reach at least the specified value

### acps-cli admin discovery dsp registry-info

Purpose: show information about the currently connected Registry.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp sync

Purpose: perform one DSP sync without resetting state.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp start

Purpose: start DSP background syncing.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp stop

Purpose: stop DSP background syncing.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp reset

Purpose: reset the DSP state without clearing already-synced data.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp hard-reset

Purpose: clear already-synced data and reset the DSP state.

Parameters: no dedicated parameters.

### acps-cli admin discovery dsp register-webhook

Purpose: register a webhook for DSP push notifications.

Parameters:

- `--url`: required, webhook callback address
- `--secret`: required, shared secret
- `--type`: the object types to subscribe to, repeatable
- `--event`: the event types to subscribe to, repeatable
- `--description`: optional description

Description: when `--type` is not provided, the subscription defaults to `acs`; when `--event` is not provided, the subscription defaults to `data_change`. This command requires a Registry administrator token, so `acps-cli admin auth login` is usually run first.

## 11. MQ admin-side commands

### 11.1 Prerequisites

`acps-cli admin mq` inherits the root-level `--config` and `--verbose`, and additionally provides:

- `--group-api-url`: overrides the mq-auth-server Group API address
- `--auth-api-url`: overrides the mq-auth-server Auth API address

Two types of certificates must be distinguished:

- Group ACL commands: require the Leader client certificate, and the certificate CN must match `--leader-aic`
- Health/Auth Probe commands: require a probe certificate usable for the mTLS handshake; it does not have to be the Leader certificate

If no certificate paths are provided in the configuration, they can be temporarily overridden with `--cert-file` and `--key-file` on each subcommand.

### acps-cli admin mq health

Purpose: probe the health status of both the mq-auth-server Group API and Auth API.

Parameters:

- `--cert-file`: overrides the probe client certificate PEM path
- `--key-file`: overrides the probe client private key PEM path
- `--json`: output the result as JSON

Description: this command is used to report health status; even if an endpoint is unreachable, it outputs `status: error` with exit code `0`, instead of treating unreachability itself as a CLI call failure.

### acps-cli admin mq group add-member

Purpose: add a member to the specified leader/group.

Parameters:

- `--leader-aic`: required, Leader Agent AIC
- `--group-id`: required, group ID
- `--member-aic`: required, the AIC of the member to add
- `--cert-file`: overrides the Leader client certificate PEM path
- `--key-file`: overrides the Leader client private key PEM path
- `--json`: output the result as JSON

### acps-cli admin mq group remove-member

Purpose: remove a member from the specified leader/group.

Parameters:

- `--leader-aic`: required, Leader Agent AIC
- `--group-id`: required, group ID
- `--member-aic`: required, the AIC of the member to remove
- `--cert-file`: overrides the Leader client certificate PEM path
- `--key-file`: overrides the Leader client private key PEM path
- `--json`: output the result as JSON

### acps-cli admin mq group delete

Purpose: delete an entire group ACL.

Parameters:

- `--leader-aic`: required, Leader Agent AIC
- `--group-id`: required, group ID
- `--yes`: skip the interactive confirmation, suitable for CI or non-TTY environments
- `--cert-file`: overrides the Leader client certificate PEM path
- `--key-file`: overrides the Leader client private key PEM path
- `--json`: output the result as JSON

Description: in a non-TTY environment `--yes` must be passed explicitly, otherwise the command is cancelled and exits with a failure.

### acps-cli admin mq group kick

Purpose: disconnect the connection of the specified member.

Parameters:

- `--leader-aic`: required, Leader Agent AIC
- `--group-id`: required, group ID
- `--member-aic`: required, the AIC of the member to kick
- `--cert-file`: overrides the Leader client certificate PEM path
- `--key-file`: overrides the Leader client private key PEM path
- `--json`: output the result as JSON

Description: if mq-auth-server cannot reach the RabbitMQ Management API, the current implementation interprets `502/503` separately as RabbitMQ Management being unreachable, rather than as an ordinary permission failure.

### acps-cli admin mq auth-probe user

Purpose: probe the `/auth/user` authorization decision.

Parameters:

- `--username`: required, the username to probe, usually an AIC
- `--cert-file`: overrides the probe client certificate PEM path
- `--key-file`: overrides the probe client private key PEM path
- `--json`: output the result as JSON

### acps-cli admin mq auth-probe vhost

Purpose: probe the `/auth/vhost` authorization decision.

Parameters:

- `--username`: required, username, usually an AIC
- `--vhost`: required, RabbitMQ vhost
- `--cert-file`: overrides the probe client certificate PEM path
- `--key-file`: overrides the probe client private key PEM path
- `--json`: output the result as JSON

### acps-cli admin mq auth-probe resource

Purpose: probe the `/auth/resource` authorization decision.

Parameters:

- `--username`: required, username, usually an AIC
- `--vhost`: required, RabbitMQ vhost
- `--resource`: required, resource type, values `exchange` or `queue`
- `--name`: required, resource name
- `--permission`: required, permission type, values `configure`, `write`, `read`
- `--cert-file`: overrides the probe client certificate PEM path
- `--key-file`: overrides the probe client private key PEM path
- `--json`: output the result as JSON

### acps-cli admin mq auth-probe topic

Purpose: probe the `/auth/topic` authorization decision.

Parameters:

- `--username`: required, username, usually an AIC
- `--vhost`: required, RabbitMQ vhost
- `--resource`: resource type, default `topic`
- `--name`: required, Exchange name
- `--permission`: required, permission type, values `write` or `read`
- `--routing-key`: required, routing key
- `--cert-file`: overrides the probe client certificate PEM path
- `--key-file`: overrides the probe client private key PEM path
- `--json`: output the result as JSON

## 12. Common command examples

```bash
uv run acps-cli --config ./acps-cli.toml auth login --username alice --password 'S3cret!'
uv run acps-cli agent save --acs-file ./acs.json --json
uv run acps-cli cert eab fetch --aic <AIC> --output ./private/eab.json
uv run acps-cli cert issue --aic <AIC> --eab-file ./private/eab.json --usage clientAuth
uv run acps-cli discover query "北京旅游推荐" --limit 5
uv run acps-cli monitor status
uv run acps-cli monitor metrics series --metric cpu.usage --aic <AIC> --start 2026-06-25T00:00:00Z --end 2026-06-25T01:00:00Z --step 1m
uv run acps-cli admin registry review list --status submitted --json
uv run acps-cli admin discovery run-sync --no-hard-reset --expect-acs-min 1
uv run acps-cli admin mq health --json
```

## 13. Suggested reading order

If this is your first contact with the project, it is recommended to read and use it in the following order:

1. First confirm the service addresses and certificate paths in `acps-cli.toml`
2. Then look at the command tree in this document to determine whether you follow the user-side or the admin-side path
3. Finally look up the specific parameters in the section for the corresponding command

If you still have questions about a command, you can use the following help to view the Click output of the currently running version:

```bash
uv run acps-cli --help
uv run acps-cli cert --help
uv run acps-cli discover query --help
uv run acps-cli monitor --help
uv run acps-cli admin mq group delete --help
```
