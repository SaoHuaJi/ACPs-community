**[English](README_en.md) | [中文](README.md)**

# Keycloak Realm Imports

This directory provides the 3 realm import templates used by the `dev-infra` Keycloak:

- `acps-registry.json`
- `acps-monitor.json`
- `acps-leader.json`

Import goals:

1. Create mutually isolated realms for the three projects.
2. Pre-provision the two base clients `<project>-web` and `<project>-api` for each realm; `../bootstrap-eddsa.sh` goes on to add Device Grant CLI clients such as `registry-cli` / `monitor-cli`.
3. Pre-provision project roles, optional OIDC scopes, and a number of test users for each realm.

Notes:

1. These JSON files mainly carry the realm, clients, roles, scopes, test users, and audience / claim mappers.
2. `redirectUris` and `webOrigins` are aligned automatically by `bootstrap-dev-keycloak.sh` (which internally calls `bootstrap-eddsa.sh`) after `dev-infra` starts, using the local development URLs (`REGISTRY_WEB_BASE_URL`, `MONITOR_WEB_BASE_URL`, `LEADER_WEB_BASE_URL`).
3. Keycloak's signing key material is runtime state; putting private key material directly into the repository is not recommended. The realm import already sets `defaultSignatureAlgorithm` to `EdDSA`, and `bootstrap-eddsa.sh` fills in the `eddsa-generated` key provider (`Ed25519`, active/enabled) for each realm. If you start Keycloak by hand, you should still confirm in the admin console or the Admin API that:
   - `Realm Settings > Tokens > Default Signature Algorithm = EdDSA`
   - `Realm Settings > Keys` contains an active/enabled `eddsa-generated` key with curve `Ed25519`
4. The `roles` client scope already explicitly maps the client roles of `<project>-api` to `resource_access.<project>-api.roles`. `bootstrap-eddsa.sh` synchronously fills in the corresponding role scope mappings, so that `fullScopeAllowed=false` can still issue access tokens containing the project roles. Do not turn `fullScopeAllowed` on to "fix" missing roles; when a new role is needed, keep adjusting within the corresponding `*-api` client roles and the mapper / scope mapping conventions.
5. The `tenant_id` / `allowed_aics` in the `acps-monitor` realm enter the access token via user attributes + mappers. If this is later switched to an authorization service or group mapping, the mappers should be adjusted accordingly.

The default password for the test users is uniformly `demo123`.

Product delivery (installation package / image package) has its own copies of the Keycloak realm templates; see `release/install-packaging/templates/keycloak/realms/` and `release/image-packaging/infra/keycloak/realms/`.
