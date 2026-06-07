# CrowdStrike AIDR TypeScript SDK Reference

## Installation

```bash
npm install @crowdstrike/aidr
# or
pnpm add @crowdstrike/aidr
# or
yarn add @crowdstrike/aidr
```

## Client Setup

```typescript
import { AIGuard } from "@crowdstrike/aidr";

const client = new AIGuard({
  token: "your-api-token",
  baseURLTemplate: "https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}",
});
```

### Constructor Options

```typescript
interface ClientOptions {
  /** API token (required) */
  token: string;

  /** URL template with {SERVICE_NAME} placeholder (required) */
  baseURLTemplate: string;

  /** Max retry attempts (default: 2) */
  maxRetries?: number;

  /** Max polling attempts for HTTP 202 responses (default: 5) */
  maxPollingAttempts?: number;

  /** Request timeout in milliseconds (default: 60000) */
  timeout?: number;

  /** Custom fetch implementation */
  fetch?: Fetch;

  /** Default headers for all requests */
  defaultHeaders?: HeadersLike;
}
```

The service name `aiguard` is set internally on the `AIGuard` class.

## Retries

Default: 2 retries with exponential backoff (0.5s initial, 8s max, 25% jitter). Retries on connection errors, HTTP 408, 409, 429, 5xx.

```typescript
// Client-level
const client = new AIGuard({
  token: "...",
  baseURLTemplate: "...",
  maxRetries: 3,
});

// Per-request override
const response = await client.guardChatCompletions(
  { guard_input: { messages: [...] } },
  { maxRetries: 5 }
);
```

## Timeouts

Default: 60 seconds (60000ms).

```typescript
// Client-level (milliseconds)
const client = new AIGuard({
  token: "...",
  baseURLTemplate: "...",
  timeout: 30000, // 30 seconds
});

// Per-request override
const response = await client.guardChatCompletions(
  { guard_input: { messages: [...] } },
  { timeout: 10000 } // 10 seconds
);
```

## Abort / Cancellation

Pass an `AbortSignal` to cancel requests:

```typescript
const controller = new AbortController();

const response = await client.guardChatCompletions(
  { guard_input: { messages: [...] } },
  { signal: controller.signal }
);

// Cancel from elsewhere
controller.abort();
```

## Async Polling (HTTP 202)

The SDK automatically polls when the API returns HTTP 202 (Accepted). Configure max polling attempts:

```typescript
const client = new AIGuard({
  token: "...",
  baseURLTemplate: "...",
  maxPollingAttempts: 10, // default: 5
});

// Per-request override
const response = await client.guardChatCompletions(
  { guard_input: { messages: [...] } },
  { maxPollingAttempts: 3 }
);
```

Set `maxPollingAttempts: 0` to disable polling and return the accepted response directly.

## Error Handling

```typescript
import { AIGuard, Errors } from "@crowdstrike/aidr";

try {
  const response = await client.guardChatCompletions({
    guard_input: { messages: [...] },
  });
} catch (error) {
  if (error instanceof Errors.APIUserAbortError) {
    console.log("Request was aborted");
  } else if (error instanceof Errors.APIConnectionError) {
    console.log("Connection error:", error.message);
    console.log("Cause:", error.cause);
  } else if (error instanceof Errors.APIError) {
    console.log("API error:", error.status, error.message);
    console.log("Headers:", error.headers);
    console.log("Error body:", error.error);
  }
}
```

### Error Classes

```
AIDRError
└── APIError<TStatus, THeaders, TError>
    ├── APIUserAbortError    (status: undefined)
    └── APIConnectionError   (status: undefined)
```

`APIError` properties:
- `status: number | undefined` — HTTP status code
- `headers: Headers | undefined` — response headers
- `error: object | undefined` — parsed error body

## Guard Chat Completions

```typescript
const response = await client.guardChatCompletions({
  guard_input: {
    messages: [
      { role: "user", content: "Your prompt here" },
    ],
  },
  event_type: "input",
});
```

### Request Body Type: ChatCompletionsGuard

```typescript
interface ChatCompletionsGuard {
  /** Required — chat messages object */
  guard_input: Record<string, unknown>;

  /** Optional fields */
  app_id?: string;
  user_id?: string;
  llm_provider?: string;
  model?: string;
  model_version?: string;
  source_ip?: string;
  source_location?: string;
  tenant_id?: string;
  event_type?: "input" | "output" | "tool_input" | "tool_output" | "tool_listing"; // default: "input"
  collector_instance_id?: string;
  input_fpe_context?: string;
  extra_info?: {
    app_name?: string;
    app_group?: string;
    app_version?: string;
    actor_name?: string;
    actor_group?: string;
    source_region?: string;
    sub_tenant?: string;
    mcp_tools?: Array<{
      server_name: string; // min length 1
      tools: string[];     // min length 1, each min length 1
    }>;
    [key: string]: unknown; // additional properties allowed
  };
}
```

### Request Options

All methods accept an optional second argument:

```typescript
interface RequestOptions {
  query?: object;
  body?: unknown;
  maxRetries?: number;
  maxPollingAttempts?: number;
  timeout?: number;
  baseURLTemplate?: string;
  headers?: HeadersLike;
  signal?: AbortSignal;
}
```

### Response Type

```typescript
type MaybeAcceptedResponse<T> = AcceptedResponse | SuccessResponse<T>;

// Success case
interface SuccessResponse<T> {
  request_id: string;
  request_time: string;
  response_time: string;
  status: "Success";
  summary: string;
  result: T;
}

// Accepted case (async processing)
interface AcceptedResponse {
  request_id: string;
  request_time: string;
  response_time: string;
  status: "Accepted";
  summary: string;
  result: {
    ttl_mins: number;
    retry_counter: number;
    location: string;
  };
}
```

The `result` for guard chat completions contains:

```typescript
{
  guard_output?: Record<string, unknown>;
  blocked?: boolean;
  transformed?: boolean;
  policy?: string;
  fpe_context?: string;
  detectors?: {
    malicious_prompt?: {
      detected?: boolean;
      data?: {
        action?: string;
        analyzer_responses?: Array<{
          analyzer: string;
          confidence: number;
        }>;
      };
    };
    confidential_and_pii_entity?: {
      detected?: boolean;
      data?: {
        entities?: Array<{
          action: string;
          type: string;
          value: string;
          start_pos?: number;
        }>;
      };
    };
    malicious_entity?: {
      detected?: boolean;
      data?: {
        entities?: Array<{
          type: string;
          value: string;
          start_pos?: number;
          raw?: Record<string, unknown>;
        }>;
      };
    };
    secret_and_key_entity?: { /* same shape as confidential_and_pii_entity */ };
    custom_entity?: { /* same shape as confidential_and_pii_entity */ };
    competitors?: {
      detected?: boolean;
      data?: { action?: string; entities?: string[]; };
    };
    language?: {
      detected?: boolean;
      data?: { action?: string; language?: string; };
    };
    topic?: {
      detected?: boolean;
      data?: {
        action?: string;
        topics?: Array<{ topic: string; confidence: number; }>;
      };
    };
    code?: {
      detected?: boolean;
      data?: { action?: string; language?: string; };
    };
  };
  access_rules?: Record<string, unknown>;
}
```

## Unredact

```typescript
const response = await client.unredact({
  fpe_context: "base64-fpe-context",
  redacted_data: { /* redacted content */ },
});
console.log(response.result?.data); // unredacted content
```

### Request Body

```typescript
{
  fpe_context: string;    // required — base64 FPE context
  redacted_data: unknown; // required — data to unredact
}
```

## Valibot Schemas

The SDK exports Valibot schemas for runtime validation:

```typescript
import {
  AidrPostV1GuardChatCompletionsResponseSchema,
  AidrPostV1UnredactResponseSchema,
  ChatCompletionsGuardSchema,
  PangeaResponseSchema,
} from "@crowdstrike/aidr/schemas/ai-guard";

import * as v from "valibot";

// Validate a response
const parsed = v.parse(AidrPostV1GuardChatCompletionsResponseSchema, response);

// Validate request body before sending
const validBody = v.parse(ChatCompletionsGuardSchema, requestBody);
```

### Available Schemas

Key schemas from `@crowdstrike/aidr/schemas/ai-guard`:

| Schema | Description |
|--------|-------------|
| `PangeaResponseSchema` | Base response schema |
| `ChatCompletionsGuardSchema` | Guard request body |
| `AidrPostV1GuardChatCompletionsResponseSchema` | Guard response |
| `AidrPostV1UnredactResponseSchema` | Unredact response |
| `AidrPromptInjectionResultSchema` | Prompt injection detector result |
| `AidrRedactEntityResultSchema` | PII/secret/custom entity result |
| `AidrMaliciousEntityResultSchema` | Malicious entity result |
| `AidrSingleEntityResultSchema` | Competitor entity result |
| `AidrLanguageResultSchema` | Language detector result |
| `AidrTopicResultSchema` | Topic detector result |
| `AidrAccessRulesResponseSchema` | Access rules result |

From `@crowdstrike/aidr/schemas`:

| Schema | Description |
|--------|-------------|
| `AcceptedResponseSchema` | HTTP 202 accepted response |
| `ErrorResponseSchema` | Error response with error codes |
| `SuccessResponseSchema(resultSchema)` | Factory for success responses |
| `APIResponseSchema(resultSchema)` | Union of accepted, success, error |

## Complete Example

```typescript
import { AIGuard } from "@crowdstrike/aidr";
import {
  AidrPostV1GuardChatCompletionsResponseSchema,
} from "@crowdstrike/aidr/schemas/ai-guard";
import * as v from "valibot";

const client = new AIGuard({
  token: "your-api-token",
  baseURLTemplate: "https://api.eu-1.crowdstrike.com/aidr/{SERVICE_NAME}",
  maxRetries: 3,
  timeout: 30000,
});

async function scanPrompt(userMessage: string) {
  const response = await client.guardChatCompletions({
    guard_input: {
      messages: [{ role: "user", content: userMessage }],
    },
    event_type: "input",
    extra_info: {
      app_name: "my-app",
    },
  });

  // Validate response schema
  await v.parseAsync(AidrPostV1GuardChatCompletionsResponseSchema, response);

  if (response.status === "Success" && response.result) {
    console.log("Blocked:", response.result.blocked);
    console.log("Transformed:", response.result.transformed);

    const detectors = response.result.detectors;

    if (detectors?.malicious_prompt?.detected) {
      console.log("Prompt injection detected!");
      for (const a of detectors.malicious_prompt.data?.analyzer_responses ?? []) {
        console.log(`  ${a.analyzer}: ${a.confidence}`);
      }
    }

    if (detectors?.confidential_and_pii_entity?.detected) {
      for (const e of detectors.confidential_and_pii_entity.data?.entities ?? []) {
        console.log(`PII: ${e.type} -> ${e.value} (${e.action})`);
      }
    }

    // Unredact if FPE was used
    if (response.result.fpe_context) {
      const unredacted = await client.unredact({
        fpe_context: response.result.fpe_context,
        redacted_data: response.result.guard_output,
      });
      console.log("Unredacted:", unredacted.result?.data);
    }
  }
}

scanPrompt("My SSN is 123-45-6789 and my key is sk-abc123");
```

> **This example shows a single guard call in isolation.** For the full integration pattern — guarding input, calling an LLM, and guarding the output — see the **Guarding Model Output**, **Tool Use Guarding**, and **Multi-Turn Conversation Guarding** sections below.

## Guarding Model Output

When guarding LLM output, construct messages with `role: "assistant"` — never use the SDK content block's `.type` field (which is `"text"`, not a valid chat role).

Use a narrow literal type for `role` to stay compatible with the Anthropic SDK's `MessageParam` type, which requires `role: "user" | "assistant"`:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { AIGuard, Errors } from "@crowdstrike/aidr";

type ChatMessage = { role: "user" | "assistant"; content: string };

// Collect text blocks from the LLM response
const outputMessages: ChatMessage[] = claudeResponse.content
  .filter((block): block is Anthropic.TextBlock => block.type === "text")
  .map((block) => ({ role: "assistant" as const, content: block.text }));

// Guard with event_type="output"
const guardResponse = await client.guardChatCompletions({
  guard_input: { messages: outputMessages },
  event_type: "output",
  extra_info: { app_name: "my-app" },
});

if (guardResponse.status !== "Success") {
  throw new Error(`Unexpected AI Guard status: ${guardResponse.status}`);
}
const result = guardResponse.result;
if (!result) {
  throw new Error("Empty result from AI Guard");
}

if (result.blocked) {
  // Return null or throw — avoid process.exit() in library/server code
  throw new Error("[AI Guard] Model output was blocked by policy.");
}

// Use transformed content if available
let safeOutput: ChatMessage[] = outputMessages;
if (result.transformed && result.guard_output) {
  const guardOut = result.guard_output as Record<string, unknown>;
  if (Array.isArray(guardOut["messages"])) {
    safeOutput = guardOut["messages"] as ChatMessage[];
  }
}

// If FPE was used, unredact before displaying to the caller
if (result.fpe_context) {
  const unredacted = await client.unredact({
    fpe_context: result.fpe_context,
    redacted_data: result.guard_output,
  });
  const data = unredacted.result?.data as Record<string, unknown> | undefined;
  if (data && Array.isArray(data["messages"])) {
    safeOutput = data["messages"] as ChatMessage[];
  }
}

// safeOutput is ChatMessage[] — role is "user" | "assistant", compatible with Anthropic MessageParam
for (const msg of safeOutput) {
  console.log(msg.content);
}
```

When passing guarded messages back to the Anthropic SDK (e.g., for multi-turn), the narrow `role` type is directly assignable to `Anthropic.MessageParam` without casting.

## Tool Use Guarding

Guard tool calls and results using the `tool_input`, `tool_output`, and `tool_listing` event types.

### Guarding Tool Inputs (before execution)

When the model generates a tool call, guard it before executing:

```typescript
import Anthropic from "@anthropic-ai/sdk";

// Build tool input messages from Anthropic tool_use blocks
const toolInputMessages = claudeResponse.content
  .filter((block): block is Anthropic.ToolUseBlock => block.type === "tool_use")
  .map((block) => ({
    role: "assistant" as const,
    content: JSON.stringify({ tool: block.name, input: block.input }),
  }));

const toolGuard = await client.guardChatCompletions({
  guard_input: { messages: toolInputMessages },
  event_type: "tool_input",
  extra_info: { app_name: "my-app" },
});

if (toolGuard.status !== "Success") {
  throw new Error(`Unexpected AI Guard status: ${toolGuard.status}`);
}
if (!toolGuard.result) {
  throw new Error("Empty result from AI Guard");
}
if (toolGuard.result.blocked) {
  console.log("[AI Guard] Tool call blocked by policy.");
  return;
}
// Execute the tool
```

### Guarding Tool Outputs (before feeding back to model)

After executing a tool, guard the result before returning it to the model:

```typescript
const toolOutputMessages = [
  { role: "user" as const, content: String(toolResult) },
];

const outGuard = await client.guardChatCompletions({
  guard_input: { messages: toolOutputMessages },
  event_type: "tool_output",
  extra_info: { app_name: "my-app" },
});

if (outGuard.status !== "Success") {
  throw new Error(`Unexpected AI Guard status: ${outGuard.status}`);
}
if (!outGuard.result) {
  throw new Error("Empty result from AI Guard");
}
if (outGuard.result.blocked) {
  console.log("[AI Guard] Tool output blocked by policy.");
  return;
}
```

### Guarding Tool Listings (MCP)

When presenting MCP tool definitions to the model, guard the listing using `event_type: "tool_listing"`. Pass tool metadata via `extra_info.mcp_tools` — this tells AI Guard which tools are being offered so policies can be applied:

```typescript
const listingGuard = await client.guardChatCompletions({
  guard_input: { tools: mcpToolDefinitions },
  event_type: "tool_listing",
  extra_info: {
    app_name: "my-app",
    mcp_tools: [
      { server_name: "my-mcp-server", tools: ["search", "query"] },
    ],
  },
});

if (listingGuard.status !== "Success") {
  throw new Error(`Unexpected AI Guard status: ${listingGuard.status}`);
}
if (!listingGuard.result) {
  throw new Error("Empty result from AI Guard");
}
if (listingGuard.result.blocked) {
  throw new Error("Tool listing blocked by AI Guard policy.");
}
```

## Multi-Turn Conversation Guarding

In multi-turn conversations, guard only the **new** message — not the full history. Store the guarded (possibly transformed) version in history so redacted content stays consistent across turns.

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { AIGuard } from "@crowdstrike/aidr";

// Both clients are assumed to be constructed at module level:
// const client = new AIGuard({ token: process.env.AIDR_TOKEN!, baseURLTemplate: "..." });
// const anthropicClient = new Anthropic(); // reads ANTHROPIC_API_KEY from env

type ChatMessage = { role: "user" | "assistant"; content: string };

const conversation: ChatMessage[] = [];

async function chat(userInput: string): Promise<void> {
  const newMessage: ChatMessage = { role: "user", content: userInput };

  // Guard only the new incoming message
  const inputGuard = await client.guardChatCompletions({
    guard_input: { messages: [newMessage] },
    event_type: "input",
    extra_info: { app_name: "my-app" },
  });

  if (inputGuard.status !== "Success") {
    throw new Error(`Unexpected AI Guard status: ${inputGuard.status}`);
  }
  if (!inputGuard.result) {
    throw new Error("Empty result from AI Guard");
  }
  if (inputGuard.result.blocked) {
    console.log("[AI Guard] Input blocked.");
    return;
  }

  // Use guarded (possibly redacted) version
  let safeInput: ChatMessage[] = [newMessage];
  if (inputGuard.result.transformed && inputGuard.result.guard_output) {
    const guardOut = inputGuard.result.guard_output as Record<string, unknown>;
    if (Array.isArray(guardOut["messages"])) {
      safeInput = guardOut["messages"] as ChatMessage[];
    }
  }

  // Append guarded message to history and call the model
  conversation.push(...safeInput);

  const llmResponse = await anthropicClient.messages.create({
    model: "claude-opus-4-6",
    max_tokens: 16000,
    messages: conversation, // ChatMessage[] is assignable to MessageParam[]
  });

  // Guard the output
  const outputMessages: ChatMessage[] = llmResponse.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => ({ role: "assistant" as const, content: b.text }));

  const outputGuard = await client.guardChatCompletions({
    guard_input: { messages: outputMessages },
    event_type: "output",
    extra_info: { app_name: "my-app" },
  });

  if (outputGuard.status !== "Success") {
    throw new Error(`Unexpected AI Guard status: ${outputGuard.status}`);
  }
  if (!outputGuard.result) {
    throw new Error("Empty result from AI Guard");
  }
  if (outputGuard.result.blocked) {
    console.log("[AI Guard] Output blocked.");
    // Roll back the user turn we just appended
    conversation.splice(conversation.length - safeInput.length);
    return;
  }

  let safeOutput: ChatMessage[] = outputMessages;
  if (outputGuard.result.transformed && outputGuard.result.guard_output) {
    const guardOut = outputGuard.result.guard_output as Record<string, unknown>;
    if (Array.isArray(guardOut["messages"])) {
      safeOutput = guardOut["messages"] as ChatMessage[];
    }
  }

  // Append the guarded assistant response to history
  conversation.push(...safeOutput);

  for (const msg of safeOutput) {
    console.log(`Assistant: ${msg.content}`);
  }
}
```

Key rules:
- Guard only the **new** message, not the entire history — avoids re-scanning and double-redacting already-processed content
- Store the **guarded (transformed) version** in history, not the original
- On output block, roll back the pending user turn to keep conversation history consistent

## Package Info

- Package: `@crowdstrike/aidr` (npm)
- Runtime dependency: `valibot ^1.2.0`
- Exports: CJS + ESM (dual format)
- License: MIT
