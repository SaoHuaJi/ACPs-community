**[English](README_en.md) | [中文](README.md)**

# ACPs SDK

Agent Collaboration Protocols (agent collaboration protocol system) SDK — the Python implementation of the ACPs protocol system.

Currently this SDK contains the following modules:

| Module         | Description                                              |
| -------------- | -------------------------------------------------------- |
| `acps_sdk.acs` | Agent Capability Specification                           |
| `acps_sdk.adp` | Agent Discovery Protocol                                 |
| `acps_sdk.aic` | Agent Identity Code                                      |
| `acps_sdk.aip` | Agent Interaction Protocol                               |

## 1. Local SDK Development Environment

### 1.1. Setting Up the Development Environment

It is recommended to use uv to install and manage Python 3.14, and to manage dependencies and builds through uv, avoiding reliance on the system Python.

```bash
# Install uv (if not already installed, the official install script is recommended)
# macOS / Linux (user-level global installation)
curl -LsSf https://astral.sh/uv/install.sh | sh
# Windows (PowerShell, user-level global installation)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Clone the repository
git clone <repository-url>
cd acps-sdk

# Install Python 3.14 with uv
uv python install 3.14

# Create and activate a virtual environment based on the uv-managed Python 3.14
uv venv --python 3.14 .venv
source .venv/bin/activate
# Windows PowerShell:
# .\.venv\Scripts\Activate.ps1

# Install the SDK runtime dependencies and development dependencies (including tests)
uv sync
```

`uv sync` synchronizes `[dependency-groups].dev` (pytest, the PyJWT required for OIDC testing, etc.). If you only want to install the runtime dependencies, you can use `uv sync --no-dev`; if the target application needs OIDC capabilities, additionally run `uv sync --extra oidc` (or `uv add 'acps-sdk[oidc]'`).

Run the tests:

```bash
uv run pytest
# or
uv run pytest tests/
```

### 1.2. Building and Publishing

```bash
uv build
```

The generated wheel and sdist will be located in the `dist/` directory.

Publish to PyPI:

```bash
uv publish --token <PyPI_TOKEN>
```

## 2. Installing the SDK in a Target Project

### 2.1 Using pip

In the Python environment where this SDK is needed, you can install it with pip.

From PyPI (after release):

```bash
pip install acps-sdk
```

From a local wheel file:

```bash
pip install path/to/acps_sdk-2.0.0-py3-none-any.whl
```

### 2.2 Using uv

In a project that needs this SDK, using uv to manage dependencies is recommended.

From PyPI (after release):

```bash
uv add acps-sdk
```

From a local wheel file:

```bash
uv add path/to/acps_sdk-2.0.0-py3-none-any.whl
```

From local source code:

```bash
uv add --editable ../acps-sdk
```

## 3. SDK Usage Example Code

```python
from acps_sdk.acs import AgentCapabilitySpec
from acps_sdk.aip import AipRpcClient, TaskState
from acps_sdk.aic import validate_aic_format, parse_aic
```
