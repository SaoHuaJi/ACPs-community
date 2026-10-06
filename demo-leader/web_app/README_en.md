**[English](README_en.md) | [中文](README.md)**

# Web App Front End

This directory is the front-end Web interface of the Demo application, implemented with vanilla JavaScript, providing an interaction interface with the Leader Agent.

## 1. Directory Structure

```
web_app/
├── index.html      # Main page
├── app.js          # Front-end logic (API calls, polling, message rendering)
├── config.js       # Runtime configuration (backend address, polling interval, etc.)
├── styles.css      # Stylesheet
└── webserver.py    # Static file server (based on the Python standard library)
```

## 2. Technology Stack

A "zero-dependency" strategy is adopted, with no build tools and no third-party runtime libraries:

- **Framework**: Vanilla JavaScript (ES2020)
- **Styling**: Hand-written CSS
- **HTTP**: Native `fetch`
- **Build**: None (served directly as static files)

## 3. Interaction Flow

1. The user enters a natural-language request in the input box.
2. The front end calls `POST /api/v1/submit` to submit the request to the Leader.
3. The front end polls the task status via `GET /api/v1/result/{session_id}`.
4. The interface is updated according to the response status (`pending` / `running` / `awaiting_input` / `completed` / `failed`).
5. Switching between the two execution modes, Direct RPC and Group, is supported.

## 4. Configuration

The default local development configuration is placed in `runtime-config.js`; `config.js` is responsible for providing fallback values and merging them with the runtime overrides. In the deployed state, the installation layer's Ansible can rewrite `runtime-config.js` through a Jinja template.

| Configuration Item | Default Value           | Description              |
| ------------------ | ----------------------- | ------------------------ |
| `backendBase`      | `http://localhost:9031` | Leader backend address   |
| `apiVersion`       | `v1`                    | API version              |
| `pollInterval`     | `5000`                  | Polling interval (ms)    |
| `maxPollRetries`   | `60`                    | Maximum polling attempts |

## 5. Running

Run from the repository root (recommended):

```bash
# Integrated local development (Leader API + Web UI)
just dev start
```

Or start the static file server separately:

```bash
uv run python web_app/webserver.py              # Default 127.0.0.1:9030
uv run python web_app/webserver.py --port 4000  # Specify port
```

After starting, visit `http://localhost:9030` in the browser.

> **Note**: The front end requires the Leader service (default `http://localhost:9031`) to be running in order to work properly. Local development has OIDC enabled by default, and points to the Keycloak provided by `acps-infra/dev-infra` via `runtime-config.js`:
> `http://localhost:9080/realms/acps-leader`.

