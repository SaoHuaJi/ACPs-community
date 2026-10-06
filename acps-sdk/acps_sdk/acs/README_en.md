**[English](README_en.md) | [中文](README.md)**

# ACS SDK — Agent Capability Specification Models

The ACS (Agent Capability Specification) module provides Python data models based on the **ACPs-spec-ACS-v02.02** specification, implemented with Pydantic V2 for type validation and serialization.

## Core Models

| Model                 | Description                                                |
| --------------------- | ---------------------------------------------------------- |
| `AgentCapabilitySpec` | ACS root object, fully describing the agent's identity, capabilities, endpoints, security schemes and skills |
| `AgentProvider`       | Agent service provider information (organization, contact details, qualifications) |
| `AgentCapabilities`   | Technical capability configuration (streaming responses, asynchronous notifications, message queues) |
| `MQProtocolVersion`   | Message queue protocol version enum (such as MQTT, AMQP, Kafka, Redis, RabbitMQ) |
| `AgentEndPoint`       | Service endpoint configuration (URL, transport protocol, security requirements) |
| `AgentSkill`          | Agent skill definition (functional boundaries, input/output specifications) |

## Quick Start

```python
from acps_sdk.acs import AgentCapabilitySpec

# Create from a dict
spec = AgentCapabilitySpec.from_dict(data)

# Create from a JSON string
spec = AgentCapabilitySpec.from_json(json_str)

# Load from a JSON file
spec = AgentCapabilitySpec.from_file("agent.json")

# Serialize
json_str = spec.to_json()
data_dict = spec.to_dict()
```

## Field Aliases

The models use the `populate_by_name=True` configuration, supporting both Python-style (`snake_case`) and protocol-style (`camelCase`) field names.

`AgentCapabilitySpec.to_json()` and `AgentCapabilitySpec.to_dict()` use `camelCase` aliases by default and exclude fields whose value is `None`. If Pydantic's `model_dump()` / `model_dump_json()` is called directly, `by_alias=True` must be passed explicitly to output protocol-style field names.

## References

- [ACPs-spec-ACS-v02.02](../../../acps-specs/03-ACPs-spec-ACS/ACPs-spec-ACS_en.md) - Agent capability description
