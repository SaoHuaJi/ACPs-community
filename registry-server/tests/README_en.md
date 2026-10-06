**[English](README_en.md) | [中文](README.md)**

# Test Directory Notes

## Layering Conventions

- `unit/`: pure unit tests, verifying only the logic of this repository, without depending on a real database or real network services.
- `integration/`: verify the integration behavior of the `registry-server` runtime itself with the test database; external CA / other sibling services must be expressed through fake peers, stub transports or contract fixtures, rather than requiring real sibling processes to run at the same time.
- `e2e/`: verify the black-box behavior of a managed, temporary `registry-server` instance, covering the public plane, the real mTLS `9002` listener, authentication, Agent lifecycle, Webhook CRUD and other self-contained capabilities of this service.

## Boundary Notes

- By default, `tests/` in this repository does not host real cross-service integration e2e that requires starting multiple sibling business services at the same time.
- For infrastructure dependencies that are necessary for the authentication boundary of `registry-server` itself and can be stably provided by the managed dev-infra (such as Keycloak), black-box integration cases may be added in this repository.
- Integration chains involving real interaction between `registry-server` and `ca-server`, `discovery-server` are uniformly verified in `acps-cli/tests/e2e/`.
- If a scenario requires starting real sibling services at the same time, sharing the `.env` of sibling repositories, or relying on cross-repository topology collaboration, then it does not belong to the `integration` / `e2e` of this repository, but to the integration test scope of `acps-cli`.

## Running Entry Points

- `just test unit`: pure unit layer.
- `just test integration`: this service + test database + fake peer.
- `just test e2e`: managed startup of a temporary `registry-server` test instance, and injects `TEST_E2E_BASE_URL`, `TEST_E2E_MTLS_BASE_URL` and the mTLS client certificate environment variables.
- `just test e2e -- tests/e2e/test_oidc_keycloak_flow.py`: run only the OIDC black-box cases; the shared entry point orchestrates by the `local -> oidc` profile, and manages the startup of Keycloak and a temporary `registry-server` test instance.
- `just test`: execute `unit -> integration -> e2e` in sequence.
- Before running black-box e2e, please first prepare the development PKI artifacts under the local `certs/` via `just prep certs` or `just test bootstrap`.
- The current goal of the black-box e2e of this repository is not to depend on sibling repository environment files; if real sibling business service integration is required, please switch to `acps-cli`.

## skip and warning Conventions

- Problems such as missing sibling services, missing manual integration environments, or missing external `.env` should not be avoided long-term through `skip`; such prerequisites should be absorbed through bootstrap, fixtures or managed test instances.
- `skip` should only be applied to future work, capabilities not yet implemented, or explicitly unsupported platform paths.
- warning is part of test health; if a warning is found, test code, fixtures or dependency compatibility issues should be fixed first, rather than ignored long-term.

## Debugging Suggestions

- To debug the black-box e2e of this repository, you can prefer `just test e2e`, letting the test entry point automatically start a temporary instance.
- To manually connect directly to `9002`, please use `certs/client.pem` / `certs/client.key` and `certs/trust-bundle.pem` to complete a real mTLS handshake.
- To debug real cross-service behavior, please switch to the `acps-cli` repository, and run according to the integration test instructions in its README.
