**[English](README_en.md) | [中文](README.md)**

# Generic Partner Agent Framework

This directory implements a generic Partner Agent framework, adopting a **"generic runtime + configuration-driven"** architecture. Adding a new Agent only requires writing Prompt and JSON configuration files, with no need to write Python code. All AIP protocol interactions, state machine transitions and exception handling are uniformly maintained by the generic framework.

## 1. Directory Structure

```
partners/
├── main.py              # Main entry point: scans the online/ directory and starts the FastAPI service
├── generic_runner.py    # Generic runtime: encapsulates the three-stage processing logic of a single Agent
├── group_handler.py     # Group mode message queue handling
├── online/              # [Active area] Agents that are running (the main program only loads this directory)
│   ├── beijing_urban/   # Beijing urban attractions planner
│   ├── beijing_rural/   # Beijing suburban attractions planner
│   ├── beijing_food/    # Beijing food recommendation specialist
│   ├── china_hotel/     # Nationwide hotel booking specialist
│   └── china_transport/ # Nationwide transportation booking specialist
└── offline/             # [Editing area] Agents under development, maintenance or paused
```

### Configuration Files of Each Agent

Each Agent directory (such as `online/beijing_urban/`) contains the following configuration files:

| File           | Description                                                      |
| -------------- | ---------------------------------------------------------------- |
| `acs.json`     | ACS definition: Agent metadata (name, AIC), capability boundary (Skills list) |
| `config.toml`  | Runtime parameters: LLM connection/Profile configuration, concurrency limits, etc. |
| `prompts.toml` | Business logic: three-stage Prompt templates (Decision / Analysis / Production) |

## 2. Core Processing Flow

The lifecycle of each Partner Agent handling a Leader request is divided into three standard stages:

1. **Intent recognition and admission decision (Decision)**: determine whether the request is within the service scope, and decide whether to accept or reject the task.
2. **Requirement analysis and completion (Analysis)**: convert natural language into structured requirement data, identify missing information and generate follow-up questions.
3. **Content generation and delivery (Production)**: call the large model based on the complete requirements to generate the final deliverable.

The specific behavior of these three stages is entirely driven by the Prompts in `prompts.toml`; different Agents only need to write different Prompts to implement different business logic.

## 3. Deployment Mode

An **independent port** mode is used: each Agent runs as an independent process on an independent port, with mTLS support. Ports and certificates are configured in each Agent's `config.toml`.

- RPC interface: `/rpc`
- Group RPC interface: `/group/rpc`
- Health check: `/health`

## 4. Running

Run from the `demo-partner` project root directory (recommended):

```bash
# Start the Partner service
just dev start
```

Or start directly through Python:

```bash
python -m partners.main
```

## 5. Basic Process for Adding an Agent

1. Create a new directory under `offline/` (such as `offline/my_agent/`).
2. Write three configuration files:
   - `acs.json`: define the Agent identity and capabilities.
   - `config.toml`: configure the LLM Profile, service port and mTLS parameters. Actual LLM values are injected through the repository root `.env`; online Partners share the two groups of variables `PARTNER_LLM_FAST_*` and `PARTNER_LLM_DEFAULT_*` by default.
   - `prompts.toml`: write the Prompts for the three stages Decision / Analysis / Production.
3. Move the directory into `online/` to bring it online:
   ```bash
   mv offline/my_agent online/my_agent
   ```
4. Restart the Partner service to make it take effect (the current version does not yet support hot reloading).
