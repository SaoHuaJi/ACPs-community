**[English](README_en.md) | [中文](README.md)**

# Demo Leader tree template (ACS / scenario)

- `atr/acs.json`: the Leader itself (usually AMQP); the installer injects advertise + AMQP host.
- `scenario/expert/*/*.json`: static Partner ACS snapshots; HTTPS is rewritten to **demo_partner**'s
  `acps_advertise_host` (not the Leader's own host, unless they are on the same machine).

Inside the template `localhost` is only a placeholder; it is rewritten at installation time by `scripts/rewrite_acs_endpoints.py`.
