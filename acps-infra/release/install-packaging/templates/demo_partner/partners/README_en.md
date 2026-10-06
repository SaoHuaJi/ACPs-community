**[English](README_en.md) | [中文](README.md)**

# Demo Partner ACS template

`https://localhost:902x` / `amqps://rabbitmq:5671` in `*/acs.json` are **placeholders waiting to be rewritten by the installer**:

- HTTPS/JSONRPC → the local `acps_advertise_host` (Public plane)
- AMQP → `rabbitmq` on the same machine in image mode; otherwise / in host mode always `acps_group_addr(rabbitmq)`

Rewrite entry point: `scripts/rewrite_acs_endpoints.py` (before the `cert_provision` bootstrap + the `demo_partner` role).
Do not treat these localhost URLs as runtime truth.
