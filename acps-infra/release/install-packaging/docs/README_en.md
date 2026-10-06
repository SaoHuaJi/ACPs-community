**[English](README_en.md) | [中文](README.md)**

# install-packaging Operations Documentation Index

> Product main path: `acps-image-install-*` / `acps-host-install-*` + Ansible `playbooks/*.yml`.  
> For concepts see [`acps-infra/README_en.md`](../../../README_en.md).

This directory is a short index inside the installation tree; for step-by-step operations, the **acps-docs** tutorials are authoritative.

## Which Document to Read

| Scenario | Document |
| --- | --- |
| Assembling the installation package | [acps-docs …/install-package-build_en.md](../../../../acps-docs/tutorials/install-package-build_en.md) |
| Local / remote single-host first install (image / host) | [acps-docs …/install-package-ansible-deploy_en.md](../../../../acps-docs/tutorials/install-package-ansible-deploy_en.md) |
| Multi-host (three-node split) | [acps-docs …/install-package-ansible-deploy-3nodes_en.md](../../../../acps-docs/tutorials/install-package-ansible-deploy-3nodes_en.md) |
| Multi-host (app + deps two hosts) | In-package `inventories/hosts-multi.example.yml` + §Multi-host below |
| OIDC Web / Device | [oidc-web-app-manual-verification_en.md](../../../../acps-docs/tutorials/oidc-web-app-manual-verification_en.md), [oidc-acps-cli-device-login_en.md](../../../../acps-docs/tutorials/oidc-acps-cli-device-login_en.md) |
| Upgrade / rollback | In this tree, the "Upgrade" / "Rollback" sections of `README_en.md` |
| Monitor / AMP | Enabled with Monitor by product default; business acceptance D see `business.yml` |
| Known limitations | [known-limitations.md](./known-limitations.md) |
| Bypass entry contract | "Bypass contract" in [known-limitations.md](./known-limitations.md) |

## Inventory Quick Reference

| File | Purpose |
| --- | --- |
| `inventories/hosts.example.yml` | image single host (`acps-node-1`) |
| `inventories/hosts.rocky9.yml` / `hosts.ubuntu22.yml` / `hosts.rocky8.yml` / `hosts.ubuntu20.yml` | host single host |
| `inventories/hosts.4os-multi.yml` + `host_vars/*.acps.local.yml` | host four-OS mixed deployment (one host per component group) |
| `inventories/hosts-multi.example.yml` | image multi-host: app + deps |
| `inventories/secrets.example.yml` | Copy to `secrets.yml` (never commit real secrets) |

```bash
cd "$PKG/ansible"   # or the source tree release/install-packaging/ansible
cp inventories/hosts-multi.example.yml inventories/hosts.yml
# Change the host names according to the comments, and write the corresponding host_vars
cp inventories/secrets.example.yml inventories/secrets.yml
# CA: scripts/generate_ca_materials.sh …
ansible-playbook -i inventories/hosts.yml playbooks/preflight.yml -e @inventories/secrets.yml
ansible-playbook -i inventories/hosts.yml playbooks/site.yml -e @inventories/secrets.yml
ansible-playbook -i inventories/hosts.yml playbooks/smoke.yml -e @inventories/secrets.yml
# After both demos are enabled:
ansible-playbook -i inventories/hosts.yml playbooks/business.yml -e @inventories/secrets.yml
```

## Half-Install Recovery (After Fail-Fast)

1. Check the play log on the control node and `journalctl -u acps-*` / `docker compose ps` on the target hosts.  
2. Rerun the failed step under the **same inventory / secrets**: preferably `ansible-playbook … playbooks/site.yml --tags <phase_tag>` (see `site.yml` tags); or a full `site.yml` (**final-state convergence**: when there are no material/configuration changes it should skip issuance and avoid unconditional restarts; **do not use site as upgrade** — for upgrade/rollback use `upgrade.yml` / `rollback.yml`, for renewal use `renew-certs.yml` / `refresh-trust-bundle.yml`).  
3. **"Run site once more" is a mandatory convergence acceptance test**, not optional; after the rerun, the key observations (active, TLS, alive-sync) should match the stable final state.  
4. **Use `--limit` with care**: rerunning only one host may leave cross-host certificate/advertise address inconsistency; after a multi-host repair it is advisable to run `smoke.yml` once more.  
5. Failure in the middle of an upgrade: keep `releases/` and state; after fixing the cause, rerun `upgrade.yml` (do not run `down -v` against production hosts).

## controller Workspace Isolation (Recommended / Mandatory for Convergence Verification)

When doing multiple sets of deployments on the same `controller` (image / host Rocky / host Ubuntu, or parallel gating), **use an independent working directory (WS) for each deployment** to avoid overwriting each other's inventory, secrets, CA and control-node state:

```text
~/acps-ws/conv-<step>-<topo>-<YYYYMMDD-HHMMSS>/
  pkg/     # Unpack root of this deployment's installation package (the only execution tree)
  logs/    # All tee logs of this process
  meta.txt
```

Unpacking, changing configuration, running `ansible-playbook` and writing logs all happen inside that WS. When target business hosts do not overlap, multiple WSs may run concurrently; it is **forbidden** for two processes to run `site` against the same business host at the same time, and sharing the same unpacked directory is also forbidden. Suggested bypass verification: `~/acps-ws/bypass-sbN-<topo>-YYYYMMDD-HHMMSS/`.

## Bypass Playbooks (Renewal / trust / Upgrade / Rollback)

| Playbook | Responsibility | Do not mix up |
| --- | --- | --- |
| `renew-certs.yml` | Leaf certificate renewal when nearing expiry / force | **Does not fix SAN**; for SAN → `site` or an explicit full force |
| `refresh-trust-bundle.yml` | Fetch and distribute trust | Does not renew leaves |
| `upgrade.yml` | Upgrade of an explicit component list | **Is not** `site.yml`; when upgrading `ca_server` the target follows refresh by default (SB9) |
| `rollback.yml` | Switch to `previous` | Does **not** roll back certificates/DB; after rollback do not overwrite back with site |
| CLI rollback (image/host) | Adapter may fork; CSV/state/previous/smoke are isomorphic | See the known-limitations BP15 contract |

The renew/refresh slice covered by running site once more only proves that the bypass entries are usable; the bypass implementations themselves have already converged per the "Bypass contract" in [known-limitations.md](./known-limitations.md).

## Monitor / AMP Enablement Prerequisites (Summary)

- When `monitor_server_enabled: true` (product default), Redpanda / VM / CH / MinIO / OpenSearch + Forwarder are installed along with it.  
- Disk and memory are assessed per site (see `group_vars` and component defaults).  
- ClickHouse / OpenSearch are currently plaintext HTTP.  
- Dual-OS / renaming a business host serial deployment: advertise/SAN fingerprint changes are automatically re-signed; issuer changes clear the leaf certificates and force re-signing; after certificates/trust land, the dependency closure is restarted; smoke/renew/refresh perform fail-closed handshake acceptance against the real TLS plane.
