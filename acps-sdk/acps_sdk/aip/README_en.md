**[English](README_en.md) | [中文](README.md)**

# AIP SDK — Agent Interaction Protocol v2

The AIP SDK provides a Python implementation of the AIP v2 protocol, covering the task command/result models, JSON-RPC interaction, RabbitMQ collaboration in group mode, and client/server helper capabilities for streaming and mTLS scenarios.

## Directory Structure

```text
acps_sdk/aip/
├── __init__.py             # Root module public exports
├── aip_base_model.py       # AIP v2 base data models
├── aip_rpc_model.py        # JSON-RPC and AIP RPC data models
├── aip_rpc_client.py       # AIP RPC client
├── aip_rpc_server.py       # FastAPI RPC server helper utilities
├── aip_stream_model.py     # Streaming transport data models
├── aip_group_model.py      # Group mode data models
├── aip_group_leader.py     # Group mode Leader-side implementation
├── aip_group_partner.py    # Group mode Partner-side implementation
├── aip_group_auth.py       # Group ACL service client
├── aip_group_runtime.py    # Group runtime naming, invitation and AMQP URL utilities
└── README.md               # This file
```

## Feature Overview

### Communication Modes

| Mode | Use Case | Transport | Main Entry Point |
| ---- | -------- | --------- | ---------------- |
| RPC mode | 1:1 Leader-Partner request/response interaction | HTTP(S) + JSON-RPC | `AipRpcClient`; the server helper utilities are in `aip_rpc_server.py` |
| Group mode | 1:N Leader coordinating collaboration among multiple Partners | RabbitMQ (AMQP) + Fanout Exchange | `GroupLeader` / `GroupLeaderMqClient` / `GroupPartnerMqClient` |
| Streaming model | Data modeling for the SSE or reconnecting streaming protocol | JSON-RPC + event model | `aip_stream_model.py` |

### Root Module Exports

`from acps_sdk.aip import ...` currently exports the following commonly used types:

| Category | Exported Objects |
| -------- | ---------------- |
| Base models | `TaskState`, `TaskCommandType`, `DataItem`, `TextDataItem`, `FileDataItem`, `StructuredDataItem`, `Message`, `TaskCommand`, `TaskResult`, `TaskStatus`, `Product`, `GetCommandParams`, `StartCommandParams` |
| RPC client | `AipRpcClient` |
| Group models | `ACSObject`, `GroupInfo`, `GroupMgmtCommandType`, `GroupMgmtCommand`, `GroupMgmtResult`, `GroupMemberStatus`, `RabbitMQRequest`, `RabbitMQResponse`, `RabbitMQRequestParams`, `RabbitMQServerConfig`, `AMQPConfig` |
| Group clients | `GroupLeaderMqClient`, `GroupLeaderSession`, `GroupLeader`, `GroupPartnerMqClient`, `PartnerGroupSession`, `PartnerGroupState` |

The RPC request/response models `RpcRequest` and `RpcResponse`, the server utilities `CommandHandlers` / `add_aip_rpc_router`, the streaming models, the mTLS utilities and some group invitation/runtime utilities are not exported from the root module; import them from the corresponding submodule.

### Task Commands and States

The Leader sends commands through `TaskCommand`, and the Partner reports task status and artifacts back through `TaskResult`.

| Type | Current Implementation |
| ---- | ---------------------- |
| Commands | `get`, `start`, `continue`, `cancel`, `complete`, `re-stream` |
| States | `accepted`, `working`, `awaiting-input`, `awaiting-completion`, `completed`, `canceled`, `failed`, `rejected` |
| Data items | Text `TextDataItem`, file `FileDataItem`, structured data `StructuredDataItem` |

A typical state flow is: `accepted -> working -> awaiting-input / awaiting-completion -> completed`, and it may also enter terminal states such as `failed`, `canceled` or `rejected`. `start` is a command, not a task state.

## Usage Examples

### RPC Client

```python
from acps_sdk.aip import AipRpcClient

client = AipRpcClient(
    partner_url="https://partner.example.com/aip/rpc",
    leader_id="1.2.156.3088.1.1.LDR001.ONT001.000001.0000",
)

result = await client.start_task(
    session_id="session-001",
    user_input="Please summarize this material",
)

await client.close()
```

### RPC Server

```python
from fastapi import FastAPI

from acps_sdk.aip.aip_rpc_server import CommandHandlers, add_aip_rpc_router

app = FastAPI()
handlers = CommandHandlers()

add_aip_rpc_router(app, endpoint="/aip/rpc", agent_handlers=handlers)
```

`aip_rpc_server.py` also provides `TaskManager` and `DefaultHandlers` for in-memory task state management and default command semantics; they are not exported from the `acps_sdk.aip` root module.

### Group Mode

The high-level Leader uses `GroupLeader` to manage the shared RabbitMQ connection, group sessions, invitations and task control:

```python
from acps_sdk.aip import ACSObject, GroupLeader

leader = GroupLeader(
    leader_aic="1.2.156.3088.1.1.LDR001.ONT001.000001.0000",
    rabbitmq_config={
        "host": "mq.example.com",
        "port": 5672,
        "vhost": "/",
        "user": "guest",
        "password": "guest",
    },
)

session = await leader.create_group_session(
    session_id="session-001",
    initial_partners=[
        ACSObject(aic="1.2.156.3088.1.1.PTR001.ONT001.000001.0000"),
    ],
)

task_id = await leader.start_task(
    session_id=session.session_id,
    task_content="Please collaborate on a research task",
)

runtime = leader.get_group_runtime(session.session_id)
await leader.dissolve_group_session(session.session_id)
await leader.close()
```

The low-level `GroupLeaderMqClient` can directly create a RabbitMQ fanout exchange, publish task commands, send management commands and dissolve groups; the corresponding Partner-side entry point is `GroupPartnerMqClient`. A Partner can handle direct RPC invitations through `join_group()`, or receive an `InboxGroupInvitation` through the inbox queue and then call `join_group_from_invitation()`.

The Partner side provides helper methods for reporting status:

```python
await partner.accept_task(task_id, session_id)
await partner.start_working(task_id, session_id)
await partner.request_input(task_id, session_id, "Please provide additional input")
await partner.submit_for_completion(task_id, session_id, products)
await partner.complete_task(task_id, session_id)
await partner.fail_task(task_id, session_id, "Reason for the failure")
```

### mTLS Configuration

```python
import ssl

client_ssl_context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
client_ssl_context.load_cert_chain(
    certfile="./certs/client.pem",
    keyfile="./certs/client.key",
)
client_ssl_context.load_verify_locations(cafile="./certs/trust-bundle.pem")
client_ssl_context.check_hostname = False  # Can be turned off for local integration testing when there is no SAN/domain name
client_ssl_context.verify_mode = ssl.CERT_REQUIRED
client_ssl_context.minimum_version = ssl.TLSVersion.TLSv1_2

server_ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
server_ssl_context.load_cert_chain(
    certfile="./certs/server.pem",
    keyfile="./certs/server.key",
)
server_ssl_context.load_verify_locations(cafile="./certs/trust-bundle.pem")
server_ssl_context.verify_mode = ssl.CERT_REQUIRED
server_ssl_context.minimum_version = ssl.TLSVersion.TLSv1_2
```

`AipRpcClient`, `AipNotificationClient`, `AipStreamClient` and the group mode related clients can all accept `ssl_context`, for HTTPS + mTLS or AMQPS EXTERNAL authentication scenarios. If you want to expose your own HTTPS / mTLS callback or RPC entry point, `server_ssl_context` needs to be handed to the HTTP/ASGI server or reverse proxy that actually carries TLS, rather than to the SDK's routing helper function itself.

## References

- [ACPs-spec-AIP-v02.02](../../../acps-specs/07-ACPs-spec-AIP/ACPs-spec-AIP_en.md) - Agent Interaction Protocol
