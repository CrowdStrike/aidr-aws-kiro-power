---
name: "crowdstrike-aidr-sdk"
displayName: "CrowdStrike AIDR SDK"
description: "SDK reference for CrowdStrike AI Defense (AIDR) across Go, Python, and TypeScript. Covers AI Guard content scanning, PII redaction, prompt injection detection, and unredact workflows."
keywords: ["crowdstrike", "aidr", "ai-guard", "ai-defense", "pii-redaction", "prompt-injection"]
author: "CrowdStrike"
---

# CrowdStrike AIDR SDK

## Overview

CrowdStrike AIDR (AI Defense & Response) provides APIs for securing AI-powered applications. The primary service is **AI Guard**, which analyzes and redacts content to prevent prompt injection, PII leakage, malicious entity injection, and other threats in LLM interactions.

SDKs are available in three languages:
- **Go** — `github.com/crowdstrike/aidr-go` (imported as `aidr`)
- **Python** — `crowdstrike-aidr` (PyPI)
- **TypeScript** — `@crowdstrike/aidr` (npm)

All three SDKs share the same API surface: a client configured with a base URL template and API token, exposing `guardChatCompletions` and `unredact` methods.

## Available Steering Files

- **go-sdk.md** — Complete Go SDK reference: installation, client setup, request options, retries, timeouts, error handling, types, and examples
- **python-sdk.md** — Complete Python SDK reference: installation, client setup, timeouts, retries, error handling, models, and examples
- **typescript-sdk.md** — Complete TypeScript SDK reference: installation, client setup, retries, timeouts, polling, schemas, and examples

Read the appropriate steering file based on the language being used.

## Core Concepts (All SDKs)

### Base URL Template

All SDKs use a URL template with a `{SERVICE_NAME}` placeholder that gets replaced at request time. For AI Guard, the service name is `aiguard`.

```
https://api.<region>.crowdstrike.com/aidr/{SERVICE_NAME}
```

Regions include `us-1`, `us-2`, `eu-1`, `us-gov-1`, etc.

### Authentication

All SDKs authenticate via a Bearer token passed in the `Authorization` header. The token is set during client construction.

### AI Guard Service

The AI Guard service provides two endpoints:

1. **Guard Chat Completions** (`POST /v1/guard_chat_completions`)
   - Analyzes chat completion messages for threats
   - Detects: prompt injection, PII, secrets/keys, malicious entities, malicious URLs/domains, competitors, language, topics, code
   - Can redact or block content based on policy
   - Supports event types: `input`, `output`, `tool_input`, `tool_output`, `tool_listing`

2. **Unredact** (`POST /v1/unredact`)
   - Decrypts FPE (Format-Preserving Encryption) redacted content
   - Requires the `fpe_context` returned from a guard call that used FPE redaction

### Guard Input Format

The `guard_input` field accepts a structured object matching the OpenAI chat completions format:

```json
{
  "messages": [
    {"role": "user", "content": "Your prompt here"}
  ]
}
```

### Response Structure

All responses follow the Pangea response schema:

```json
{
  "request_id": "prq_...",
  "request_time": "2024-01-01T00:00:00Z",
  "response_time": "2024-01-01T00:00:01Z",
  "status": "Success",
  "summary": "...",
  "result": {
    "blocked": false,
    "transformed": false,
    "policy": "default",
    "guard_output": { ... },
    "fpe_context": "...",
    "detectors": {
      "malicious_prompt": { "detected": false, "data": { ... } },
      "confidential_and_pii_entity": { "detected": false, "data": { ... } },
      "malicious_entity": { "detected": false, "data": { ... } },
      "secret_and_key_entity": { "detected": false, "data": { ... } },
      "custom_entity": { "detected": false, "data": { ... } },
      "competitors": { "detected": false, "data": { ... } },
      "language": { "detected": false, "data": { ... } },
      "topic": { "detected": false, "data": { ... } },
      "code": { "detected": false, "data": { ... } }
    },
    "access_rules": { ... }
  }
}
```

> **All fields are optional/nullable.** `result`, `detectors`, and every individual detector object may be `null`/`None`. Always check for `None` before accessing nested fields. Never access `response.result.blocked`, `response.result.detectors`, or `detectors.<name>.detected` without first confirming the parent is not `None`.

### Detectors

| Detector | Description |
|----------|-------------|
| `malicious_prompt` | Prompt injection / jailbreak attempts. Returns analyzer responses with confidence scores. |
| `confidential_and_pii_entity` | PII like SSNs, emails, phone numbers. Returns entities with type, value, action, start_pos. |
| `malicious_entity` | Malicious URLs, domains, IPs. Returns entities with type, value, raw data. |
| `secret_and_key_entity` | API keys, tokens, passwords. Returns entities with type, value, action. |
| `custom_entity` | User-defined entity patterns. Returns entities with type, value, action. |
| `competitors` | Competitor brand mentions. Returns entity list and action. |
| `language` | Language detection. Returns detected language and action. |
| `topic` | Topic classification. Returns topics with confidence scores. |
| `code` | Code detection. Returns detected language and action. |

### Extra Info / MCP Tool Listing

When using AI Guard with MCP-based agentic workflows, you can pass MCP tool metadata via `extra_info.mcp_tools`:

```json
{
  "extra_info": {
    "app_name": "my-agent",
    "mcp_tools": [
      {
        "server_name": "my-mcp-server",
        "tools": ["tool_a", "tool_b"]
      }
    ]
  }
}
```

### Retries & Timeouts

All SDKs default to 2 retries with exponential backoff (0.5s initial, 8s max, 25% jitter). Retries trigger on connection errors, HTTP 408, 409, 429, and 5xx responses.

### Async / Accepted Responses

The API may return HTTP 202 with status `"Accepted"` for long-running requests. The TypeScript SDK handles polling automatically. For Go and Python, check the response status and poll the async endpoint if needed.

## Quick Start Examples

### Go
```go
client := aidr.NewClient(
    option.WithBaseURLTemplate("https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}"),
    option.WithToken("your-token"),
)
resp, err := client.AIGuard.GuardChatCompletions(ctx, aidr.AIGuardGuardChatCompletionsParams{
    GuardInput: map[string]any{"messages": []any{map[string]any{"role": "user", "content": "Hello"}}},
    EventType:  aidr.AIGuardGuardChatCompletionsParamsEventTypeInput,
})
```

### Python
```python
from crowdstrike_aidr import AIGuard

client = AIGuard(
    base_url_template="https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}",
    token="your-token",
)
response = client.guard_chat_completions(
    guard_input={"messages": [{"role": "user", "content": "Hello"}]},
    event_type="input",
)
```

### TypeScript
```typescript
import { AIGuard } from "@crowdstrike/aidr";

const client = new AIGuard({
    baseURLTemplate: "https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}",
    token: "your-token",
});
const response = await client.guardChatCompletions({
    guard_input: { messages: [{ role: "user", content: "Hello" }] },
    event_type: "input",
});
```

## Auto-Install Hook

When adding AI Guard to a Python project, create a Kiro hook to automatically install the SDK dependency whenever a Python file importing `crowdstrike_aidr` is saved. This ensures the package is always available without manual intervention.

Hook configuration:
- Event: `fileEdited` on `**/*.py`
- Action: `askAgent` — check if the file contains `import crowdstrike_aidr` or `from crowdstrike_aidr`, and if so run `pip install crowdstrike-aidr`

Similarly for TypeScript (`npm install @crowdstrike/aidr` on `**/*.ts`) or Go (`go get github.com/crowdstrike/aidr-go` on `**/*.go`).

## Troubleshooting

### "Connection refused" or timeout
- Verify the base URL template region matches your CrowdStrike tenant
- Ensure `{SERVICE_NAME}` placeholder is present in the template
- Check network connectivity and firewall rules

### 401 Unauthorized
- Verify your API token is valid and not expired
- Ensure the token has permissions for the AIDR service

### 400 Bad Request
- Check that `guard_input` contains the expected key: `messages` for `input`, `output`, `tool_input`, and `tool_output` events; `tools` for `tool_listing` events
- Ensure `event_type` is one of: `input`, `output`, `tool_input`, `tool_output`, `tool_listing`

### Empty detector results
- Detectors are configured per-policy; ensure your policy has the relevant detectors enabled
- Check the `policy` field in the response to confirm which policy was applied

## License and Support

MIT License

Copyright (c) 2026 CrowdStrike

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

### Support

Issues may be reported on [GitHub](https://github.com/CrowdStrike/aidr-aws-kiro-power/issues) and are used to track bugs, documentation updates, enhancement requests, and security concerns.

### Support Escalation

We endeavor to provide support for `@crowdstrike/aidr-aws-kiro-power` within the GitHub repository. This expands our online knowledge base, enables self-help for our community, and can reduce the time needed to receive answers.

If you are a CrowdStrike customer and would prefer to have your questions or issues handled directly with CrowdStrike Support, you are welcome to contact the CrowdStrike technical support team.
