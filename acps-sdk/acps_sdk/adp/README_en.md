**[English](README_en.md) | [中文](README.md)**

# ADP (Agent Discovery Protocol) SDK Module

This module implements the shared functionality code of ADP (Agent Discovery Protocol) in the ACPs protocol system, based on the ACPs-spec-ADP protocol specification.

## Module Structure

```
acps_sdk/adp/
├── __init__.py      # Package entry point, uniformly exports all public APIs
├── constants.py     # Protocol constants (defaults, limit parameters, API paths, etc.)
├── errors.py        # Error code enums, exception classes and error response building utilities
├── models.py        # Pydantic V2 data models (requests, responses, filters, context, etc.)
├── validators.py    # Validation utilities (forward chain, fan-out budget, filter conditions, etc.)
└── README.md        # This file
```

## Feature Overview

### Data Models (`models.py`)

Implemented with Pydantic V2:

| Model Class           | Description                                                |
| --------------------- | ---------------------------------------------------------- |
| `DiscoveryRequest`    | Discovery request, containing query type, filter conditions and forward control parameters |
| `DiscoveryResponse`   | Discovery response, following the CommonResponse pattern where result/error are mutually exclusive |
| `DiscoveryResult`     | Discovery result, containing acsMap, the agents candidate list and optional routes |
| `DiscoveryRoute`      | Route-level result, corresponding to one forwarding path and its candidates |
| `DiscoveryAgentGroup` | Agent matching results organized by group                  |
| `DiscoveryAgentSkill` | Matching result of a single agent skill                    |
| `DiscoveryFilter`     | Structured filter condition collection (supports nesting)  |
| `FilterCondition`     | A single filter condition                                  |
| `FilterOperator`      | Filter operator enum                                       |
| `DiscoveryContext`    | Context payload                                            |
| `ErrorDetail`         | Error information object                                   |

Request and response body fields use lowerCamelCase via Pydantic aliases; snake_case can continue to be used in Python code. `DiscoveryRequest` and `DiscoveryResponse` provide `to_json()` / `to_dict()`, which output camelCase by default and exclude `None` fields; if Pydantic's `model_dump()` is called directly, `by_alias=True` must be passed explicitly.

| Convenience Method/Property                      | Description                               |
| ------------------------------------------------ | ----------------------------------------- |
| `DiscoveryRequest.from_dict()` / `from_json()`   | Build a request object from a dict or JSON |
| `DiscoveryRequest.get_effective_*()`             | Get the effective values of depth, fan-out and timeout (including defaults) |
| `DiscoveryResponse.success()` / `failure()`      | Build a success or failure response       |
| `DiscoveryResponse.is_success()` / `is_error()`  | Determine the response type               |
| `DiscoveryResponse.get_adp_error()`              | Convert the `error` field into an `ADPError` |
| `DiscoveryResult.iter_agent_skills()`            | Iterate over the result skills and associate `acsMap` data by AIC |

### Error Handling (`errors.py`)

- `ADPErrorCode` enum: defines all ADP error codes (30701-50001)
- `ADPError` exception class: can be converted into a response body structure
- `make_error_response()` factory function: quickly builds an error response dict
- `get_http_status_for_error()`: gets the HTTP status code corresponding to an error code
- `ADP_ERROR_NAMES` mapping: mapping from error codes to protocol error names
- `ADP_ERROR_HTTP_STATUS` mapping: mapping from error codes to HTTP status codes

`ADPErrorCode` and `ADPError` both provide the classification methods `is_redirect()`, `is_retryable()`, `is_client_error()` and `is_forward_error()`.

### Validation Utilities (`validators.py`)

| Function                       | Description                                                          |
| ------------------------------ | -------------------------------------------------------------------- |
| `validate_discovery_request()` | Validate request parameters (type/depth/fan-out/filter nesting)      |
| `validate_forward_chain()`     | Forward chain validation (loop detection, depth check, previous-hop integrity check) |
| `validate_fanout_budget()`     | Validate whether `forwardFanoutRemaining` is sufficient; falls back to the fan-out limit when unset |
| `validate_trusted_target()`    | Check whether the target AIC is in `forwardTrustedServers`; when unset it is treated as unrestricted |
| `should_continue_forwarding()` | Determine whether the total remaining timeout is sufficient to cover a single-hop timeout |
| `build_forwarded_request()`    | Build a forwarded request (append to the chain, update the remaining budget, adjust the total timeout, optionally append signature/trust list) |
| `allocate_fanout_budget()`     | Fan-out budget allocation (supports even and weighted strategies)    |

### Constants (`constants.py`)

| Constant                           | Default Value | Description                       |
| ---------------------------------- | ------------- | --------------------------------- |
| `FORWARD_DEPTH_LIMIT_DEFAULT`      | 3             | Default forward depth             |
| `FORWARD_DEPTH_LIMIT_MAX`          | 5             | Absolute upper limit of forward depth |
| `FORWARD_DEPTH_LIMIT_MIN`          | 1             | Minimum forward depth             |
| `FORWARD_FANOUT_LIMIT_DEFAULT`     | 1             | Default forward fan-out           |
| `FORWARD_FANOUT_LIMIT_MAX`         | 5             | Absolute upper limit of forward fan-out |
| `FORWARD_FANOUT_LIMIT_MIN`         | 1             | Minimum forward fan-out           |
| `FORWARD_EACH_TIMEOUT_MS_DEFAULT`  | 10000         | Default single-hop timeout (ms)   |
| `FORWARD_TOTAL_TIMEOUT_MS_DEFAULT` | 60000         | Default total timeout (ms)        |
| `MAX_REDIRECT_HOPS`                | 5             | Maximum number of consecutive redirects |
| `FILTER_MAX_NESTING_DEPTH`         | 3             | Recommended upper limit of filter nesting depth |
| `CONTEXT_MAX_PAYLOAD_BYTES`        | 2048          | Recommended maximum context bytes |
| `DISCOVER_API_PATH`                | `/discover`   | Discovery API path                |

In addition, it exports the request header constants `HEADER_TRACE_ID`, `HEADER_SPAN_ID`, `HEADER_PARENT_SPAN_ID`, the query type constants `QUERY_TYPE_EXPLICIT` / `QUERY_TYPE_EXPLORATORY` / `QUERY_TYPE_TRENDING` / `QUERY_TYPE_FILTERED` / `QUERY_TYPES`, the route status constants `ROUTE_STATUS_OK` / `ROUTE_STATUS_TIMEOUT` / `ROUTE_STATUS_ERROR`, and the filter logic constants `FILTER_LOGIC_AND` / `FILTER_LOGIC_OR` / `FILTER_LOGIC_NOT`.

## Usage Examples

### Building a Discovery Request

```python
from acps_sdk.adp import (
    DiscoveryRequest,
    DiscoveryFilter,
    FilterCondition,
    FilterOperator,
    DiscoveryContext,
)

# Explicit query + filter conditions
request = DiscoveryRequest(
    type="explicit",
    query="I need an agent that can recommend Beijing food",
    limit=5,
    filter=DiscoveryFilter(
        conditions=[
            FilterCondition(field="active", op=FilterOperator.EQ, value=True),
            FilterCondition(
                field="endPoints.transport",
                op=FilterOperator.IN,
                value=["JSONRPC"],
            ),
            FilterCondition(
                field="capabilities.streaming",
                op=FilterOperator.EQ,
                value=True,
            ),
            FilterCondition(
                field="skills.tags",
                op=FilterOperator.ANY_OF,
                value=["food", "Beijing"],
            ),
        ]
    ),
    context=DiscoveryContext(
        conversation_id="conv-123",
        recent_turns=["The user prefers healthy food"],
        user_profile={"city": "Beijing", "budget": "medium"},
    ),
)

# Serialize to JSON (using camelCase)
json_str = request.to_json()

# It can also be serialized to a dict or deserialized from a camelCase dict
payload = request.to_dict()
same_request = DiscoveryRequest.from_dict(payload)
```

### Building a Discovery Response

```python
from acps_sdk.adp import (
    DiscoveryResponse,
    DiscoveryResult,
    DiscoveryRoute,
    DiscoveryAgentGroup,
    DiscoveryAgentSkill,
)

response = DiscoveryResponse.success(
    result=DiscoveryResult(
        acs_map={
            "AIC-AGENT-FOOD-01": {
                "name": "BeijingFoodGuide",
                "description": "Beijing food recommendation agent",
            }
        },
        agents=[
            DiscoveryAgentGroup(
                group="Food recommendations",
                agent_skills=[
                    DiscoveryAgentSkill(
                        aic="AIC-AGENT-FOOD-01",
                        skill_id="skill-beijing-food",
                        ranking=1,
                    ),
                ],
            )
        ],
        routes=[
            DiscoveryRoute(
                forward_chain=["AIC-DS-A"],
                agent_groups=[
                    DiscoveryAgentGroup(
                        group="Food recommendations",
                        agent_skills=[
                            DiscoveryAgentSkill(
                                aic="AIC-AGENT-FOOD-01",
                                skill_id="skill-beijing-food",
                                ranking=1,
                            ),
                        ],
                    )
                ],
                status="ok",
                duration_ms=150,
            )
        ],
    )
)

for aic, acs_data, skill, group in response.result.iter_agent_skills():
    print(f"[{group}] {aic} -> skill={skill.skill_id}, acs={acs_data.get('name')}")
```

### Forward Chain Validation and Construction

```python
from acps_sdk.adp import (
    DiscoveryRequest,
    validate_forward_chain,
    validate_fanout_budget,
    validate_trusted_target,
    build_forwarded_request,
    allocate_fanout_budget,
    ADPError,
)

request = DiscoveryRequest.from_dict({
    "type": "explicit",
    "query": "translation agent",
    "forwardDepthLimit": 3,
    "forwardFanoutLimit": 4,
    "forwardChain": ["AIC-DS-A"],
})

# Validate the forward chain
try:
    validate_forward_chain(
        request,
        current_server_aic="AIC-DS-B",
        sender_aic="AIC-DS-A",
    )
except ADPError as e:
    print(f"Forward chain validation failed: {e.to_error_body()}")

# Validate the fan-out budget
# When request does not set forwardFanoutRemaining, forwardFanoutLimit is used as the effective remaining budget
validate_fanout_budget(request, required_branches=3)

# Allocate the budget
budgets = allocate_fanout_budget(total_remaining=4, branch_count=3)
# -> [0, 0, 1]

# Build the forwarded request
forwarded = build_forwarded_request(
    original=request,
    current_server_aic="AIC-DS-B",
    fanout_remaining_for_branch=budgets[0],
    elapsed_ms=200,
    trusted_servers=["AIC-DS-B", "AIC-DS-C", "AIC-DS-D"],
)

print(forwarded.forward_chain)  # ["AIC-DS-A", "AIC-DS-B"]
print(forwarded.forward_fanout_remaining)  # 0
```

### Error Handling

```python
from acps_sdk.adp import (
    DiscoveryResponse,
    ADPError,
    ADPErrorCode,
    make_error_response,
    get_http_status_for_error,
)

# Using the factory function
error_body = make_error_response(
    ADPErrorCode.FORWARD_FANOUT_EXCEEDED,
    message="Fan-out budget exhausted",
    data={"availableBudget": 1, "requiredBranches": 3},
)
http_status = get_http_status_for_error(ADPErrorCode.FORWARD_FANOUT_EXCEEDED)  # 508

# Using the exception class
try:
    raise ADPError(
        code=ADPErrorCode.FORWARD_LOOP_DETECTED,
        message="Forward loop detected",
        data={"forwardChain": ["AIC-A", "AIC-B", "AIC-A"]},
    )
except ADPError as e:
    http_status = e.http_status  # 508
    response_body = e.to_response_dict()
    print(e.is_forward_error())  # True

error_response = DiscoveryResponse.failure(40001, "MissingQuery")
adp_error = error_response.get_adp_error()
if adp_error and adp_error.is_client_error():
    print(f"Client parameter error: {adp_error.message}")
```

## References

- [ACPs-spec-ADP-v02.02](../../../acps-specs/06-ACPs-spec-ADP/ACPs-spec-ADP_en.md) - Agent discovery process
