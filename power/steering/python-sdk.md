# CrowdStrike AIDR Python SDK Reference

## Installation

Requires Python v3.12 or greater.

```bash
pip install crowdstrike-aidr
```

## Auto-Install

When integrating the AIDR SDK into a project, always ensure the dependency is installed. After adding `crowdstrike_aidr` imports to a Python file, run:

```bash
pip install crowdstrike-aidr
```

To automate this, create a Kiro hook (event: `fileEdited`, pattern: `**/*.py`) that detects `crowdstrike_aidr` imports and runs `pip install crowdstrike-aidr` automatically.

## Client Setup

```python
from crowdstrike_aidr import AIGuard

client = AIGuard(
    base_url_template="https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}",
    token="your-api-token",
)
```

### Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_url_template` | `str` | required | URL template with `{SERVICE_NAME}` placeholder |
| `token` | `str` | required | API bearer token |
| `max_retries` | `int` | `2` | Max retry attempts |
| `timeout` | `float \| httpx.Timeout` | `60s (5s connect)` | Request timeout |
| `http_client` | `httpx.Client \| None` | `None` | Custom httpx client |
| `custom_headers` | `Mapping[str, str] \| None` | `None` | Extra default headers |
| `custom_query` | `Mapping[str, object] \| None` | `None` | Extra default query params |

The service name `aiguard` is set internally on the `AIGuard` class.

## Timeouts

Default: 60 seconds total, 5 seconds connect.

```python
import httpx

# Simple timeout (total seconds)
client = AIGuard(
    base_url_template="...",
    token="...",
    timeout=30.0,
)

# Granular timeout
client = AIGuard(
    base_url_template="...",
    token="...",
    timeout=httpx.Timeout(timeout=60.0, connect=10.0),
)

# Per-request timeout override
response = client.guard_chat_completions(
    guard_input={"messages": [...]},
    timeout=120.0,
)
```

## Retries

Default: 2 retries with exponential backoff (0.5s initial, 8s max). Retries on connection errors, timeouts, HTTP 408, 409, 429, 5xx.

```python
client = AIGuard(
    base_url_template="...",
    token="...",
    max_retries=5,
)
```

## Error Handling

```python
from crowdstrike_aidr import (
    AIGuard,
    APIConnectionError,
    APITimeoutError,
    APIStatusError,
    BadRequestError,
    AuthenticationError,
    PermissionDeniedError,
    NotFoundError,
    ConflictError,
    UnprocessableEntityError,
    RateLimitError,
    InternalServerError,
)

try:
    response = client.guard_chat_completions(guard_input={...})
except APITimeoutError:
    print("Request timed out")
except APIConnectionError:
    print("Connection failed")
except BadRequestError as e:
    print(f"400: {e.message}")
    print(f"Body: {e.body}")
except AuthenticationError as e:
    print(f"401: {e.message}")
except RateLimitError as e:
    print(f"429: {e.message}")
except InternalServerError as e:
    print(f"5xx: {e.message}")
except APIStatusError as e:
    print(f"HTTP {e.status_code}: {e.message}")
```

### Error Hierarchy

```
CrowdStrikeAidrError
└── APIError
    ├── APIConnectionError
    │   └── APITimeoutError
    ├── APIResponseValidationError
    └── APIStatusError
        ├── BadRequestError (400)
        ├── AuthenticationError (401)
        ├── PermissionDeniedError (403)
        ├── NotFoundError (404)
        ├── ConflictError (409)
        ├── UnprocessableEntityError (422)
        ├── RateLimitError (429)
        └── InternalServerError (5xx)
```

All `APIStatusError` subclasses have:
- `message: str` — error description
- `response: httpx.Response` — the HTTP response
- `status_code: int` — HTTP status code
- `body: object | None` — parsed JSON body or raw text

## Guard Chat Completions

```python
response = client.guard_chat_completions(
    guard_input={
        "messages": [
            {"role": "user", "content": "Your prompt here"}
        ]
    },
    event_type="input",
)
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `guard_input` | `object` | Yes | Chat messages in OpenAI format |
| `app_id` | `str \| Omit` | No | Source application/agent ID |
| `collector_instance_id` | `str \| Omit` | No | AIDR collector instance ID |
| `event_type` | `Literal["input","output","tool_input","tool_output","tool_listing"] \| Omit` | No | Event type |
| `extra_info` | `ExtraInfo \| Omit` | No | Logging schema with app metadata |
| `input_fpe_context` | `str \| Omit` | No | FPE context from a prior guard call; decrypts redacted content before scanning (multi-turn use) |
| `llm_provider` | `str \| Omit` | No | LLM provider name (e.g. "OpenAI") |
| `model` | `str \| Omit` | No | Model name (e.g. "gpt") |
| `model_version` | `str \| Omit` | No | Model version (e.g. "3.5") |
| `source_ip` | `str \| Omit` | No | User/app IP address |
| `source_location` | `str \| Omit` | No | User/app location |
| `span_id` | `str \| Omit` | No | Distributed tracing span ID |
| `tenant_id` | `str \| Omit` | No | Multi-tenant ID |
| `user_id` | `str \| Omit` | No | User/service account ID |
| `extra_headers` | `Headers \| None` | No | Additional request headers |
| `extra_query` | `Query \| None` | No | Additional query parameters |
| `extra_body` | `Body \| None` | No | Additional JSON body fields |
| `timeout` | `float \| httpx.Timeout \| None \| NotGiven` | No | Request timeout override |

### ExtraInfo Model

```python
from crowdstrike_aidr.models.ai_guard import ExtraInfo

extra_info = ExtraInfo(
    app_name="my-app",
    app_group="my-group",
    app_version="1.0.0",
    actor_name="user@example.com",
    actor_group="admin",
    source_region="us-east-1",
    sub_tenant="tenant-123",
)
```

### Response Type: GuardChatCompletionsResponse

The response is a Pydantic model. Access fields directly:

> **Nullability:** `result`, `result.detectors`, and every individual detector field are all `Optional`. Always check for `None` before accessing nested fields.

```python
response = client.guard_chat_completions(guard_input={...})

# Always check status before accessing result
if response.status != "Success":
    if response.status == "Accepted":
        raise RuntimeError("AI Guard returned Accepted — increase timeout or implement polling; see Async / 202 Accepted section")
    raise RuntimeError(f"Unexpected AI Guard status: {response.status}")

# result is Optional — guard first
if not response.result:
    raise RuntimeError("Empty result from AI Guard")

print(response.result.blocked)
print(response.result.transformed)
print(response.result.policy)

# detectors is Optional
detectors = response.result.detectors
if detectors:
    # Each detector field is also Optional — check before accessing .detected
    if detectors.malicious_prompt and detectors.malicious_prompt.detected:
        for analyzer in detectors.malicious_prompt.data.analyzer_responses:
            print(f"Analyzer: {analyzer.analyzer}, Confidence: {analyzer.confidence}")

    if detectors.confidential_and_pii_entity and detectors.confidential_and_pii_entity.detected:
        for entity in detectors.confidential_and_pii_entity.data.entities:
            print(f"PII: type={entity.type} value={entity.value} action={entity.action}")
```

## Unredact

```python
response = client.unredact(
    fpe_context="base64-fpe-context-from-guard-response",
    redacted_data={"messages": [...]},  # the redacted content
)
# result is Optional — guard before access
if not response.result:
    raise RuntimeError("Empty result from unredact")
print(response.result.data)  # unredacted content
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fpe_context` | `str` | Yes | Base64 FPE context from guard response |
| `redacted_data` | `object` | Yes | Data to unredact |
| `extra_headers` | `Headers \| None` | No | Additional request headers |
| `extra_query` | `Query \| None` | No | Additional query parameters |
| `extra_body` | `Body \| None` | No | Additional JSON body fields |
| `timeout` | `float \| httpx.Timeout \| None \| NotGiven` | No | Request timeout override |

## Async / 202 Accepted

The API may return HTTP 202 with `status: "Accepted"` for long-running requests. Always check `response.status` before accessing `result` — attempting to read `result.blocked` on an Accepted response will fail because the result shape differs.

**Recommended: increase the client timeout** to avoid receiving `"Accepted"` in most cases:

```python
client = AIGuard(
    base_url_template="...",
    token="...",
    timeout=120.0,  # raise from default 60s
)
```

**If you do receive `"Accepted"`**, the result contains a `location` URL to poll and a `ttl_mins` deadline. The Python SDK does not expose a first-class polling method; poll the location URL directly:

```python
import time
import httpx
from crowdstrike_aidr import AIGuard, APIStatusError
from crowdstrike_aidr.models import GuardChatCompletionsResponse

def guard_with_polling(client: AIGuard, token: str, params) -> GuardChatCompletionsResponse:
    response = client.guard_chat_completions(**params)

    if response.status == "Accepted":
        # Access accepted-result fields via attribute access or raw dict
        location = getattr(response.result, "location", None)
        ttl_mins = getattr(response.result, "ttl_mins", 5)
        if not location:
            raise RuntimeError("Accepted response missing location URL")

        deadline = time.monotonic() + (ttl_mins * 60)
        while time.monotonic() < deadline:
            time.sleep(5)
            poll = httpx.get(location, headers={"Authorization": f"Bearer {token}"})
            poll.raise_for_status()
            data = poll.json()
            if data.get("status") == "Success":
                # Re-parse into the typed response model
                return GuardChatCompletionsResponse.model_validate(data)
        raise RuntimeError("AI Guard polling timed out")

    if response.status != "Success":
        raise RuntimeError(f"Unexpected AI Guard status: {response.status}")

    return response
```

## Omit Sentinel

Use `omit` to explicitly exclude optional parameters:

```python
from crowdstrike_aidr import omit

response = client.guard_chat_completions(
    guard_input={...},
    app_id=omit,        # explicitly excluded
    event_type="input",  # included
)
```

## NotGiven Sentinel

`NotGiven` is used internally for parameters where `None` has a meaningful value (e.g., timeout=None means no timeout). You generally don't need to use it directly.

## Response Models

All response models are Pydantic `BaseModel` subclasses from `crowdstrike_aidr.models`:

- `PangeaResponse` — base response with `request_id`, `request_time`, `response_time`, `status`, `summary`
- `GuardChatCompletionsResponse` — extends PangeaResponse with typed `result`
- `UnredactResponse` — extends PangeaResponse with `result.data`

## Complete Example

```python
from crowdstrike_aidr import AIGuard, APIStatusError
from crowdstrike_aidr.models.ai_guard import ExtraInfo

client = AIGuard(
    base_url_template="https://api.eu-1.crowdstrike.com/aidr/{SERVICE_NAME}",
    token="your-api-token",
    max_retries=3,
    timeout=30.0,
)

try:
    response = client.guard_chat_completions(
        guard_input={
            "messages": [
                {
                    "role": "user",
                    "content": "My SSN is 123-45-6789 and my API key is sk-abc123",
                }
            ]
        },
        event_type="input",
        extra_info=ExtraInfo(app_name="my-app"),
    )
except APIStatusError as e:
    print(f"API error {e.status_code}: {e.message}")
    raise

# Always check status before accessing result (status check raises RuntimeError, not APIStatusError)
if response.status != "Success":
    raise RuntimeError(f"Unexpected AI Guard status: {response.status}")

# result is Optional — always guard before access
if not response.result:
    raise RuntimeError("Empty result from AI Guard")

print(f"Blocked: {response.result.blocked}")
print(f"Transformed: {response.result.transformed}")

# detectors is Optional — guard before iterating
detectors = response.result.detectors
if detectors:
    # Each detector field is also Optional — check before accessing .detected
    if detectors.confidential_and_pii_entity and detectors.confidential_and_pii_entity.detected:
        for entity in detectors.confidential_and_pii_entity.data.entities:
            print(f"PII detected: {entity.type} -> {entity.value} (action: {entity.action})")

    if detectors.secret_and_key_entity and detectors.secret_and_key_entity.detected:
        for entity in detectors.secret_and_key_entity.data.entities:
            print(f"Secret detected: {entity.type} -> {entity.value} (action: {entity.action})")

# Unredact if FPE was used
if response.result.fpe_context:
    unredacted = client.unredact(
        fpe_context=response.result.fpe_context,
        redacted_data=response.result.guard_output,
    )
    if not unredacted.result:
        raise RuntimeError("Empty result from unredact")
    print(f"Unredacted: {unredacted.result.data}")
```

> **This example shows a single guard call in isolation.** For the full integration pattern — guarding input, calling an LLM, and guarding the output — see the **Guarding Model Output**, **Tool Use Guarding**, and **Multi-Turn Conversation Guarding** sections below.

## Guarding Model Output

When guarding model output, construct messages with `role: "assistant"` — never use the SDK content block's `.type` field (which is `"text"`, not a valid chat role).

```python
# Collect text blocks from the LLM response
output_messages = [
    {"role": "assistant", "content": block.text}
    for block in llm_response.content
    if block.type == "text"
]

# Guard with event_type="output"
guard_response = client.guard_chat_completions(
    guard_input={"messages": output_messages},
    event_type="output",
    extra_info=ExtraInfo(app_name="my-app"),
)

# Always check status before accessing result
if guard_response.status != "Success":
    raise RuntimeError(f"Unexpected AI Guard status: {guard_response.status}")

if not guard_response.result:
    raise RuntimeError("Empty result from AI Guard")

if guard_response.result.blocked:
    print("[AI Guard] Model output blocked by policy.")
else:
    # Use transformed content if available, otherwise original
    if guard_response.result.transformed and guard_response.result.guard_output:
        guard_out = guard_response.result.guard_output
        if isinstance(guard_out, dict) and "messages" in guard_out:
            output_messages = guard_out["messages"]

    # If FPE was used, unredact before displaying to the caller
    if guard_response.result.fpe_context:
        unredacted = client.unredact(
            fpe_context=guard_response.result.fpe_context,
            redacted_data=guard_response.result.guard_output,
        )
        if not unredacted.result:
            raise RuntimeError("Empty result from unredact")
        guard_out = unredacted.result.data
        if isinstance(guard_out, dict) and "messages" in guard_out:
            output_messages = guard_out["messages"]

    for msg in output_messages:
        print(msg.get("content", ""))
```

## Tool Use Guarding

Guard tool calls and results using the `tool_input`, `tool_output`, and `tool_listing` event types.

### Guarding Tool Inputs (before execution)

When the model generates a tool call, guard it before executing. Serialize the tool call into messages:

```python
import json

# Build tool input messages from Anthropic tool_use blocks
tool_input_messages = [
    {
        "role": "assistant",
        "content": json.dumps({"tool": block.name, "input": block.input}),
    }
    for block in llm_response.content
    if block.type == "tool_use"
]

guard_response = client.guard_chat_completions(
    guard_input={"messages": tool_input_messages},
    event_type="tool_input",
    extra_info=ExtraInfo(app_name="my-app"),
)

if guard_response.status != "Success":
    raise RuntimeError(f"Unexpected AI Guard status: {guard_response.status}")
if not guard_response.result:
    raise RuntimeError("Empty result from AI Guard")

if guard_response.result.blocked:
    # Do not execute the tool
    print("[AI Guard] Tool call blocked by policy.")
    return
```

### Guarding Tool Outputs (before feeding back to model)

After executing a tool, guard the result before returning it to the model:

```python
tool_output_messages = [
    {"role": "user", "content": str(tool_result)}
]

guard_response = client.guard_chat_completions(
    guard_input={"messages": tool_output_messages},
    event_type="tool_output",
    extra_info=ExtraInfo(app_name="my-app"),
)

if guard_response.status != "Success":
    raise RuntimeError(f"Unexpected AI Guard status: {guard_response.status}")
if not guard_response.result:
    raise RuntimeError("Empty result from AI Guard")

if guard_response.result.blocked:
    print("[AI Guard] Tool output blocked by policy.")
    return

# Use transformed content if available
if guard_response.result.transformed and guard_response.result.guard_output:
    guard_out = guard_response.result.guard_output
    if isinstance(guard_out, dict) and "messages" in guard_out:
        tool_output_messages = guard_out["messages"]
```

### Guarding Tool Listings (MCP)

When presenting MCP tool definitions to the model, guard the listing using `event_type="tool_listing"`. Pass tool metadata via `extra_info.mcp_tools` — this tells AI Guard which tools are being offered so policies can be applied against tool names and server origins:

```python
from crowdstrike_aidr.models.ai_guard import ExtraInfo

guard_response = client.guard_chat_completions(
    guard_input={"tools": mcp_tool_definitions},  # your tool schema list
    event_type="tool_listing",
    extra_info=ExtraInfo(
        app_name="my-app",
        mcp_tools=[
            {"server_name": "my-mcp-server", "tools": ["search", "query"]},
        ],
    ),
)

if guard_response.status != "Success":
    raise RuntimeError(f"Unexpected AI Guard status: {guard_response.status}")
if not guard_response.result:
    raise RuntimeError("Empty result from AI Guard")

if guard_response.result.blocked:
    # Do not present these tools to the model
    raise RuntimeError("Tool listing blocked by AI Guard policy.")
```

## Multi-Turn Conversation Guarding

In multi-turn conversations, guard only the **new** message — not the full history. Store the guarded (possibly transformed) version in history so redacted content stays consistent across turns.

```python
import anthropic
from crowdstrike_aidr.models.ai_guard import ExtraInfo

# Both clients are assumed to be constructed before the loop:
# aidr_client = build_aidr_client()  (see Client Setup section)
# anthropic_client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

conversation: list[dict] = []

while True:
    user_input = input("You: ")
    new_message = {"role": "user", "content": user_input}

    # Guard only the new incoming message
    guard_response = client.guard_chat_completions(
        guard_input={"messages": [new_message]},
        event_type="input",
        extra_info=ExtraInfo(app_name="my-app"),
    )

    if guard_response.status != "Success":
        raise RuntimeError(f"Unexpected AI Guard status: {guard_response.status}")
    if not guard_response.result:
        raise RuntimeError("Empty result from AI Guard")

    if guard_response.result.blocked:
        print("[AI Guard] Input blocked.")
        continue

    # Use the guarded (possibly redacted) version
    safe_input = [new_message]
    if guard_response.result.transformed and guard_response.result.guard_output:
        guard_out = guard_response.result.guard_output
        if isinstance(guard_out, dict) and "messages" in guard_out:
            safe_input = guard_out["messages"]

    # Append guarded message to history and call the model
    conversation.extend(safe_input)

    llm_response = anthropic_client.messages.create(
        model="claude-opus-4-6",
        max_tokens=16000,
        messages=conversation,
    )

    # Guard the output
    output_messages = [
        {"role": "assistant", "content": block.text}
        for block in llm_response.content
        if block.type == "text"
    ]

    out_guard = client.guard_chat_completions(
        guard_input={"messages": output_messages},
        event_type="output",
        extra_info=ExtraInfo(app_name="my-app"),
    )

    if out_guard.status != "Success":
        raise RuntimeError(f"Unexpected AI Guard status: {out_guard.status}")
    if not out_guard.result:
        raise RuntimeError("Empty result from AI Guard")

    if out_guard.result.blocked:
        print("[AI Guard] Output blocked.")
        # Remove the user turn we just appended to keep history consistent
        conversation = conversation[:-len(safe_input)]
        continue

    safe_output = output_messages
    if out_guard.result.transformed and out_guard.result.guard_output:
        guard_out = out_guard.result.guard_output
        if isinstance(guard_out, dict) and "messages" in guard_out:
            safe_output = guard_out["messages"]

    # Append the guarded assistant response to history
    conversation.extend(safe_output)

    for msg in safe_output:
        print(f"Assistant: {msg.get('content', '')}")
```

Key rules:
- Guard only the **new** message, not the entire history — avoids re-scanning and double-redacting already-processed content
- Store the **guarded (transformed) version** in history, not the original
- On output block, roll back the pending user turn to keep conversation history consistent

## Package Info

- Package: `crowdstrike-aidr` (PyPI)
- Python: 3.12+
- Dependencies: `httpx ~=0.28.1`, `pydantic ~=2.12.5`, `typing-extensions ~=4.15.0`
- License: MIT
