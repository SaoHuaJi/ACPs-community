**[English](README_en.md) | [中文](README.md)**

# Web App Front End

This directory is the front-end Web interface of the Demo application, implemented in vanilla JavaScript, providing an interface for interacting with the Leader Agent.

## 1. Directory Structure

```
web_app/
├── index.html # main page
├── app.js # front-end logic (API calls, polling, message rendering)
├── config.js # runtime configuration (backend address, polling interval, etc.)
├── styles.css # stylesheet
└── webserver.py # static file server (based on the Python standard library)
```

## 2. Technology Stack

A "zero dependency" strategy is adopted, with no build tooling and no third-party runtime libraries:

- **Framework**: vanilla JavaScript (ES2020)
- **Styling**: hand-written CSS
- **HTTP**: native `fetch`
- **Build**: none (served directly as static files)

## 3. Interaction Flow

1. The user enters a natural-language request in the input box.
2. The front end calls `POST /api/v1/submit` to submit the request to the Leader.
3. The front end polls the task status via `GET /api/v1/result/{session_id}`.
4. The interface is updated according to the response status (`pending` / `running` / `awaiting_input` / `completed` / `failed`).
5. Switching between the two execution modes, Direct RPC and Group, is supported.

## 4. Configuration

The default local development configuration lives in `runtime-config.js`; `config.js` provides fallback values and merges them with the runtime overrides. In the deployed state, the
installation layer Ansible can rewrite `runtime-config.js` through a Jinja template.

| Configuration item | Default value | Description |
| ---------------- | ----------------------- | ---------------- |
| `backendBase` | `http://localhost:9031` | Leader backend address |
| `apiVersion` | `v1` | API version |
| `pollInterval` | `5000` | Polling interval (milliseconds) |
| `maxPollRetries` | `60` | Maximum number of polls |

## 5. Running

Run from the repository root (recommended):

```bash
# Integrated local development (Leader API + Web UI)
just dev start
```

Or start the static file server on its own:

```bash
uv run python web_app/webserver.py # default 127.0.0.1:9030
uv run python web_app/webserver.py --port 4000 # specify the port
```

After starting, visit `http://localhost:9030` in the browser.

> **Note**: The front end requires the Leader service (default `http://localhost:9031`) to be running in order to work properly. Local development enables
> OIDC by default and points through `runtime-config.js` to the Keycloak provided by `acps-infra/dev-infra`:
> `http://localhost:9080/realms/acps-leader`.
