**[English](README_en.md) | [中文](README.md)**

# acps-cli Tests

The tests of `acps-cli` are divided into three layers:

- `tests/unit/`: pure local unit tests, using mock/stub, without depending on external services.
- `tests/integration/`: verify CLI arguments, configuration, output, and the command contract for a single service.
- `tests/e2e/`: verify real cross-service workflows, state propagation, and multi-service integration topologies.

Among them, the monitor-related tests adopt a two-layer complementary approach:

- `tests/integration/test_monitor_cli_live.py`: real monitor-server + direct-write read-model setup, used to verify the contract between the CLI and the Query API.
- `tests/e2e/test_monitor_query_workflow.py`: real CLI + real writer chain, used to verify the full Kafka / storage / Query API chain.

Boundary principles:

- As long as a scenario mainly verifies the CLI's own behavior, or only involves the real interface contract of a single service, it should go into `tests/integration/`.
- As long as a scenario needs to start or chain multiple sibling services at the same time, verifying a cross-service journey or state propagation, it should go into `tests/e2e/`.

Running conventions:

```bash
just test unit
```

- `unit` can be run directly, without additional preparation.

```bash
just test bootstrap
just test integration
just test e2e
just test
```

- `integration`, `e2e` and `all` will implicitly execute `just test bootstrap` as needed; if you need to warm up the environment in advance, you can still run it once manually.
- The infra and schema required by monitor live integration / e2e will be prepared on demand according to the unified rules of `tests/_local_services.py` when pytest needs to manage the startup of monitor-server; if you choose to host monitor-server manually, please start it in the same `just dev start` way as other business services described in the README, and override the corresponding test environment variables.
- If the current shell does not set `DOCKER_CONFIG`, the test bootstrap and local automatic hosting will instead use `acps-cli/.tmp/docker-public-config/` to pull public dev-infra images, avoiding the Docker credential helper getting stuck.
- When using the default local addresses, if the target service has not been started yet, the test fixtures will automatically host the required sibling services according to the established rules.
- For more complete environment preparation, default ports and automatic hosting instructions, please refer to the "Tests" section of [../README_en.md](../README_en.md).
