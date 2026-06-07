# CrowdStrike AIDR Go SDK Reference

## Installation

Requires Go v1.24 or higher.

```bash
go get github.com/crowdstrike/aidr-go
```

```go
import (
    "github.com/crowdstrike/aidr-go"       // imported as `aidr`
    "github.com/crowdstrike/aidr-go/option"
)
```

## Client Setup

```go
client := aidr.NewClient(
    option.WithBaseURLTemplate("https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}"),
    option.WithToken("your-api-token"),
)
```

### Environment Variables

The Go SDK client can read configuration from the environment automatically — if you don't pass `WithBaseURLTemplate` or `WithToken`, the client reads:
- `AIDR_BASE_URL_TEMPLATE` — base URL template
- `AIDR_API_TOKEN` — API token

> **Note:** These names differ from the `AIDR_BASE_URL` / `AIDR_TOKEN` convention used in Python and TypeScript examples. If you need consistent env var names across all three languages (e.g., in a shared deployment config), read them explicitly and pass them as options:

```go
// Explicit read — use AIDR_BASE_URL and AIDR_TOKEN for cross-language consistency
baseURL := os.Getenv("AIDR_BASE_URL")
if baseURL == "" {
    baseURL = "https://api.us-1.crowdstrike.com/aidr/{SERVICE_NAME}"
}
token := os.Getenv("AIDR_TOKEN")
if token == "" {
    log.Fatal("AIDR_TOKEN environment variable is not set.")
}
client := aidr.NewClient(
    option.WithBaseURLTemplate(baseURL),
    option.WithToken(token),
)
```

Alternatively, rely on the SDK's native names and set `AIDR_BASE_URL_TEMPLATE` / `AIDR_API_TOKEN` in your environment.

### Client Structure

```go
type Client struct {
    Options []option.RequestOption
    AIGuard AIGuardService
}
```

The `AIGuard` field provides access to all AI Guard methods. The service name `aiguard` is set automatically.

## Request Options (Functional Options Pattern)

Options can be set at client level or per-request. Per-request options override client options.

```go
// Client-level option
client := aidr.NewClient(
    option.WithHeader("X-Custom", "value"),
    option.WithMaxRetries(3),
)

// Per-request override
resp, err := client.AIGuard.GuardChatCompletions(ctx, params,
    option.WithHeader("X-Custom", "override"),
    option.WithMaxRetries(5),
    option.WithRequestTimeout(20*time.Second),
)
```

### Available Options

| Option | Description |
|--------|-------------|
| `option.WithBaseURLTemplate(url)` | Set base URL template with `{SERVICE_NAME}` placeholder |
| `option.WithToken(token)` | Set API token |
| `option.WithServiceToken(service, token)` | Set service-specific token (overrides client token) |
| `option.WithMaxRetries(n)` | Max retry attempts (default: 2, set 0 to disable) |
| `option.WithRequestTimeout(dur)` | Per-retry timeout |
| `option.WithHeader(key, value)` | Set request header |
| `option.WithHeaderAdd(key, value)` | Append to request header |
| `option.WithHeaderDel(key)` | Remove request header |
| `option.WithQuery(key, value)` | Set query parameter |
| `option.WithJSONSet(key, value)` | Set JSON body field (sjson path syntax) |
| `option.WithJSONDel(key)` | Remove JSON body field |
| `option.WithHTTPClient(client)` | Custom `*http.Client` |
| `option.WithMiddleware(fn)` | Add request middleware |
| `option.WithResponseInto(&resp)` | Capture `*http.Response` |
| `option.WithResponseBodyInto(dst)` | Custom deserialization target |
| `option.WithDebugLog(logger)` | Log request/response (dev only) |

## Retries

Default: 2 retries with exponential backoff. Retries on connection errors, HTTP 408, 409, 429, 5xx.

```go
// Disable retries
client := aidr.NewClient(option.WithMaxRetries(0))

// Per-request retries
client.AIGuard.GuardChatCompletions(ctx, params, option.WithMaxRetries(5))
```

## Timeouts

No default timeout. Use `context.WithTimeout` for request lifecycle timeout, or `option.WithRequestTimeout` for per-retry timeout.

```go
// Overall timeout (includes all retries)
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
defer cancel()

// Per-retry timeout
client.AIGuard.GuardChatCompletions(ctx, params,
    option.WithRequestTimeout(20*time.Second),
)
```

## Error Handling

API errors return `*aidr.Error` (aliased from `apierror.Error`):

```go
resp, err := client.AIGuard.GuardChatCompletions(ctx, params)
if err != nil {
    var apierr *aidr.Error
    if errors.As(err, &apierr) {
        fmt.Println("Status:", apierr.StatusCode)
        fmt.Println(string(apierr.DumpRequest(true)))
        fmt.Println(string(apierr.DumpResponse(true)))
    }
    // Non-API errors (e.g., *url.Error wrapping *net.OpError) are returned unwrapped
    log.Fatal(err) // always terminate — do not fall through to status check
}

// Always check resp.Status before accessing resp.Result
if resp.Status != "Success" {
    // "Accepted" means the request is still processing — see Async / 202 Accepted section
    log.Fatalf("unexpected AI Guard status: %s", resp.Status)
}
```

### Error Type

```go
type Error struct {
    StatusCode int
    Request    *http.Request
    Response   *http.Response
    JSON       struct {
        ExtraFields map[string]respjson.Field
        raw         string
    }
}
```

Methods: `Error() string`, `RawJSON() string`, `DumpRequest(body bool) []byte`, `DumpResponse(body bool) []byte`

## Guard Chat Completions

```go
params := aidr.AIGuardGuardChatCompletionsParams{
    GuardInput: map[string]any{
        "messages": []any{
            map[string]any{"role": "user", "content": "Your prompt here"},
        },
    },
    EventType: aidr.AIGuardGuardChatCompletionsParamsEventTypeInput,
}

resp, err := client.AIGuard.GuardChatCompletions(ctx, params)
```

### Full Parameters

```go
type AIGuardGuardChatCompletionsParams struct {
    GuardInput          any                // required — chat messages object
    AppID               param.Opt[string]  // optional — source app/agent ID
    CollectorInstanceID param.Opt[string]  // optional — AIDR collector instance
    InputFpeContext     param.Opt[string]  // optional — FPE context from prior guard call; decrypts redacted content before scanning (multi-turn use)
    LlmProvider         param.Opt[string]  // optional — e.g. "OpenAI"
    Model               param.Opt[string]  // optional — e.g. "gpt"
    ModelVersion        param.Opt[string]  // optional — e.g. "3.5"
    SourceIP            param.Opt[string]  // optional — user/app IP
    SourceLocation      param.Opt[string]  // optional — user/app location
    TenantID            param.Opt[string]  // optional — multi-tenant ID
    UserID              param.Opt[string]  // optional — user/service account
    EventType           AIGuardGuardChatCompletionsParamsEventType // "input"|"output"|"tool_input"|"tool_output"|"tool_listing"
    ExtraInfo           AIGuardGuardChatCompletionsParamsExtraInfo // logging schema
}
```

### Event Types

```go
const (
    AIGuardGuardChatCompletionsParamsEventTypeInput       = "input"
    AIGuardGuardChatCompletionsParamsEventTypeOutput      = "output"
    AIGuardGuardChatCompletionsParamsEventTypeToolInput   = "tool_input"
    AIGuardGuardChatCompletionsParamsEventTypeToolOutput  = "tool_output"
    AIGuardGuardChatCompletionsParamsEventTypeToolListing = "tool_listing"
)
```

### ExtraInfo (Logging Schema)

```go
type AIGuardGuardChatCompletionsParamsExtraInfo struct {
    ActorGroup   param.Opt[string]
    ActorName    param.Opt[string]
    AppGroup     param.Opt[string]
    AppName      param.Opt[string]
    AppVersion   param.Opt[string]
    SourceRegion param.Opt[string]
    SubTenant    param.Opt[string]
    McpTools     []AIGuardGuardChatCompletionsParamsExtraInfoMcpTool
    ExtraFields  map[string]any  // additional arbitrary fields
}

type AIGuardGuardChatCompletionsParamsExtraInfoMcpTool struct {
    ServerName string   // required
    Tools      []string // required
}
```

### Response Type

```go
type AIGuardGuardChatCompletionsResponse struct {
    Result AIGuardGuardChatCompletionsResponseResult
    PangeaResponse // embeds RequestID, RequestTime, ResponseTime, Status, Summary
}

type AIGuardGuardChatCompletionsResponseResult struct {
    Detectors   AIGuardGuardChatCompletionsResponseResultDetectors
    AccessRules any
    Blocked     bool
    FpeContext  string // base64 — pass to Unredact if FPE was used
    GuardOutput any
    Policy      string
    Transformed bool
}
```

### Reading Response Fields

Use the `JSON` metadata struct to check field presence:

```go
if resp.Result.JSON.Blocked.Valid() {
    fmt.Println("Blocked:", resp.Result.Blocked)
}
```

## Unredact

```go
resp, err := client.AIGuard.Unredact(ctx, aidr.AIGuardUnredactParams{
    FpeContext:   fpeContextFromGuardResponse, // base64 string
    RedactedData: redactedContent,             // the redacted data to decrypt
})
// resp.Result.Data contains the unredacted data
```

## Async / 202 Accepted

The API may return HTTP 202 with `Status: "Accepted"` for long-running requests. Always check `resp.Status` before accessing `resp.Result` — the result fields are zero-value (not meaningful) on an Accepted response.

**Recommended: use `context.WithTimeout`** with a generous deadline to avoid receiving Accepted in most cases (see Timeouts section).

**If you do receive `"Accepted"`**, use `GetAsyncRequest` to poll until the result is ready:

```go
func guardWithPolling(
    ctx context.Context,
    client aidr.Client,
    params aidr.AIGuardGuardChatCompletionsParams,
) (aidr.AIGuardGuardChatCompletionsResponse, error) {
    resp, err := client.AIGuard.GuardChatCompletions(ctx, params)
    if err != nil {
        return resp, err
    }

    if resp.Status == "Accepted" {
        requestID := resp.RequestID
        // Poll every 5 seconds until Success or context deadline
        ticker := time.NewTicker(5 * time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return resp, fmt.Errorf("AI Guard polling timed out: %w", ctx.Err())
            case <-ticker.C:
                resp, err = client.AIGuard.GetAsyncRequest(ctx, requestID)
                if err != nil {
                    return resp, err
                }
                if resp.Status == "Success" {
                    return resp, nil
                }
            }
        }
    }

    if resp.Status != "Success" {
        return resp, fmt.Errorf("unexpected AI Guard status: %s", resp.Status)
    }
    return resp, nil
}
```

## Helper Functions

```go
aidr.String("value")    // param.Opt[string]
aidr.Int(42)            // param.Opt[int64]
aidr.Bool(true)         // param.Opt[bool]
aidr.Float(3.14)        // param.Opt[float64]
aidr.Time(t)            // param.Opt[time.Time]
aidr.Opt(v)             // param.Opt[T] for any comparable T
aidr.Ptr(v)             // *T
```

## Complete Example

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/crowdstrike/aidr-go"
    "github.com/crowdstrike/aidr-go/option"
)

func main() {
    // Always set a timeout — the Go SDK has no default
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    client := aidr.NewClient(
        option.WithBaseURLTemplate("https://api.eu-1.crowdstrike.com/aidr/{SERVICE_NAME}"),
        option.WithToken("your-api-token"),
    )

    resp, err := client.AIGuard.GuardChatCompletions(ctx, aidr.AIGuardGuardChatCompletionsParams{
        GuardInput: map[string]any{
            "messages": []any{
                map[string]any{
                    "role":    "user",
                    "content": "My SSN is 123-45-6789 and my email is user@example.com",
                },
            },
        },
        EventType: aidr.AIGuardGuardChatCompletionsParamsEventTypeInput,
        ExtraInfo: aidr.AIGuardGuardChatCompletionsParamsExtraInfo{
            AppName: aidr.String("my-app"),
        },
    })
    if err != nil {
        log.Fatal(err)
    }

    if resp.Status != "Success" {
        log.Fatalf("Unexpected AI Guard status: %s", resp.Status)
    }
    fmt.Printf("Blocked: %v\n", resp.Result.Blocked)
    fmt.Printf("Transformed: %v\n", resp.Result.Transformed)

    if resp.Result.Detectors.ConfidentialAndPiiEntity.Detected {
        for _, e := range resp.Result.Detectors.ConfidentialAndPiiEntity.Data.Entities {
            fmt.Printf("PII: type=%s value=%s action=%s\n", e.Type, e.Value, e.Action)
        }
    }

    if resp.Result.FpeContext != "" {
        unredacted, err := client.AIGuard.Unredact(ctx, aidr.AIGuardUnredactParams{
            FpeContext:   resp.Result.FpeContext,
            RedactedData: resp.Result.GuardOutput,
        })
        if err != nil {
            log.Fatal(err)
        }
        fmt.Printf("Unredacted: %v\n", unredacted.Result.Data)
    }
}
```

> **This example shows a single guard call in isolation.** For the full integration pattern — guarding input, calling an LLM, and guarding the output — see the **Guarding Model Output**, **Tool Use Guarding**, and **Multi-Turn Conversation Guarding** sections below.

## Guarding Model Output

When guarding LLM output, construct messages with `role: "assistant"` — never use the SDK content block's `.Type` field (which is `"text"`, not a valid chat role).

The `guardMessages` helper should follow this return convention:
- `([]any, nil)` — allowed; use the returned messages (may be redacted)
- `(nil, nil)` — blocked by policy; do not display the response
- `(nil, err)` — guard API error; suppress the response for safety

```go
// guardMessages runs messages through CrowdStrike AI Guard.
// Returns (messages, nil) if allowed, (nil, nil) if blocked, (nil, err) on error.
func guardMessages(
    ctx context.Context,
    aidrClient aidr.Client,
    messages []any,
    eventType aidr.AIGuardGuardChatCompletionsParamsEventType,
) ([]any, error) {
    resp, err := aidrClient.AIGuard.GuardChatCompletions(ctx, aidr.AIGuardGuardChatCompletionsParams{
        GuardInput: map[string]any{"messages": messages},
        EventType:  eventType,
        ExtraInfo: aidr.AIGuardGuardChatCompletionsParamsExtraInfo{
            AppName: aidr.String("my-app"),
        },
        LlmProvider: aidr.String("Anthropic"),                // set to the provider generating the content being scanned
        Model:       aidr.String("claude-opus-4-6"),         // set to the model actually generating the content being scanned
    })
    if err != nil {
        return nil, err
    }

    if resp.Status != "Success" {
        return nil, fmt.Errorf("unexpected AI Guard status: %s", resp.Status)
    }

    if resp.Result.Blocked {
        return nil, nil // nil, nil signals blocked
    }

    // If content was transformed (redacted), use the sanitised version
    if resp.Result.Transformed && resp.Result.GuardOutput != nil {
        if guardOut, ok := resp.Result.GuardOutput.(map[string]any); ok {
            if msgs, ok := guardOut["messages"].([]any); ok {
                return msgs, nil
            }
        }
    }

    return messages, nil
}
```

Collect output messages from the LLM response using `role: "assistant"`:

```go
// Collect text blocks — use role "assistant", not block.Type ("text")
var outputMessages []any
for _, block := range claudeResponse.Content {
    if text, ok := block.AsAny().(anthropic.TextBlock); ok {
        outputMessages = append(outputMessages, map[string]any{
            "role":    "assistant",
            "content": text.Text,
        })
    }
}

safeOutput, err := guardMessages(ctx, aidrClient, outputMessages, aidr.AIGuardGuardChatCompletionsParamsEventTypeOutput)
if err != nil {
    // Output guard failure: suppress the response rather than leak unscanned content
    fmt.Println("[AI Guard] Could not scan model output. Response suppressed for safety.")
    return
}
if safeOutput == nil {
    fmt.Println("[AI Guard] Model output was blocked by policy.")
    return
}

for _, m := range safeOutput {
    if msg, ok := m.(map[string]any); ok {
        fmt.Println(msg["content"])
    }
}
```

### FPE / Unredact

If the guard response includes an `FpeContext`, call `Unredact` to recover the original content for trusted display:

```go
resp, err := aidrClient.AIGuard.GuardChatCompletions(ctx, params)
if err != nil { ... }
if resp.Status != "Success" { ... }

if resp.Result.FpeContext != "" {
    unredacted, err := aidrClient.AIGuard.Unredact(ctx, aidr.AIGuardUnredactParams{
        FpeContext:   resp.Result.FpeContext,
        RedactedData: resp.Result.GuardOutput,
    })
    if err != nil { ... }
    // unredacted.Result.Data contains the original content
    fmt.Printf("Unredacted: %v\n", unredacted.Result.Data)
}
```

When converting guarded `[]any` messages back to `[]anthropic.MessageParam` for multi-turn use, type-assert each entry:

```go
var anthropicMessages []anthropic.MessageParam
for _, m := range safeMessages {
    msg, ok := m.(map[string]any)
    if !ok {
        continue
    }
    content, _ := msg["content"].(string)
    role, _ := msg["role"].(string)
    if role == "user" {
        anthropicMessages = append(anthropicMessages, anthropic.NewUserMessage(anthropic.NewTextBlock(content)))
    } else if role == "assistant" {
        anthropicMessages = append(anthropicMessages, anthropic.NewAssistantMessage(anthropic.NewTextBlock(content)))
    }
}
```

> **Note on Detectors:** In Go, `Detectors` and its sub-fields are value types (not pointers). Undetected fields default to Go zero values (`false`, empty string, nil slice) — no nil check is needed before accessing `resp.Result.Detectors.MaliciousPrompt.Detected`. This differs from Python and TypeScript where every detector field is `Optional`/nullable.

## Tool Use Guarding

Guard tool calls and results using the `tool_input`, `tool_output`, and `tool_listing` event types.

### Guarding Tool Inputs (before execution)

When the model generates a tool call, guard it before executing:

```go
var toolInputMessages []any
for _, block := range llmResponse.Content {
    if toolUse, ok := block.AsAny().(anthropic.ToolUseBlock); ok {
        inputJSON, _ := json.Marshal(toolUse.Input)
        toolInputMessages = append(toolInputMessages, map[string]any{
            "role":    "assistant",
            "content": fmt.Sprintf(`{"tool":"%s","input":%s}`, toolUse.Name, inputJSON),
        })
    }
}

safeToolInput, err := guardMessages(ctx, aidrClient, toolInputMessages,
    aidr.AIGuardGuardChatCompletionsParamsEventTypeToolInput)
if err != nil {
    log.Printf("tool input guard error: %v", err)
    return
}
if safeToolInput == nil {
    fmt.Println("[AI Guard] Tool call blocked by policy.")
    return
}
// Execute the tool
```

### Guarding Tool Outputs (before feeding back to model)

After executing a tool, guard the result before returning it to the model:

```go
toolOutputMessages := []any{
    map[string]any{"role": "user", "content": toolResult},
}

safeToolOutput, err := guardMessages(ctx, aidrClient, toolOutputMessages,
    aidr.AIGuardGuardChatCompletionsParamsEventTypeToolOutput)
if err != nil || safeToolOutput == nil {
    fmt.Println("[AI Guard] Tool output blocked or guard error.")
    return
}
```

### Guarding Tool Listings (MCP)

When presenting MCP tool definitions to the model, guard the listing using `EventTypeToolListing`. Pass tool metadata via `ExtraInfo.McpTools` — this tells AI Guard which tools are being offered so policies can be applied:

```go
resp, err := aidrClient.AIGuard.GuardChatCompletions(ctx, aidr.AIGuardGuardChatCompletionsParams{
    GuardInput: map[string]any{"tools": mcpToolDefinitions},
    EventType:  aidr.AIGuardGuardChatCompletionsParamsEventTypeToolListing,
    ExtraInfo: aidr.AIGuardGuardChatCompletionsParamsExtraInfo{
        AppName: aidr.String("my-app"),
        McpTools: []aidr.AIGuardGuardChatCompletionsParamsExtraInfoMcpTool{
            {ServerName: "my-mcp-server", Tools: []string{"search", "query"}},
        },
    },
})
if err != nil { ... }
if resp.Status != "Success" { ... }
if resp.Result.Blocked {
    log.Fatal("Tool listing blocked by AI Guard policy.")
}
```

## Multi-Turn Conversation Guarding

In multi-turn conversations, guard only the **new** message — not the full history. Store the guarded (possibly transformed) version in history so redacted content stays consistent across turns.

```go
var conversation []any

for {
    fmt.Print("You: ")
    var userInput string
    fmt.Scanln(&userInput)

    newMessage := map[string]any{"role": "user", "content": userInput}

    // Guard only the new incoming message
    safeInput, err := guardMessages(ctx, aidrClient, []any{newMessage},
        aidr.AIGuardGuardChatCompletionsParamsEventTypeInput)
    if err != nil {
        log.Printf("input guard error: %v", err)
        continue
    }
    if safeInput == nil {
        fmt.Println("[AI Guard] Input blocked.")
        continue
    }

    // Append the guarded (possibly redacted) message to history
    conversation = append(conversation, safeInput...)

    // Convert []any history to []anthropic.MessageParam
    var anthropicMessages []anthropic.MessageParam
    for _, m := range conversation {
        if msg, ok := m.(map[string]any); ok {
            content, _ := msg["content"].(string)
            role, _ := msg["role"].(string)
            if role == "user" {
                anthropicMessages = append(anthropicMessages, anthropic.NewUserMessage(anthropic.NewTextBlock(content)))
            } else if role == "assistant" {
                anthropicMessages = append(anthropicMessages, anthropic.NewAssistantMessage(anthropic.NewTextBlock(content)))
            }
        }
    }
    llmResponse, err := anthropicClient.Messages.New(ctx, anthropic.MessageNewParams{
        Model:     anthropic.ModelClaudeOpus4_6,
        MaxTokens: 16000,
        Messages:  anthropicMessages,
    })
    if err != nil { ... }

    // Guard the output
    var outputMessages []any
    for _, block := range llmResponse.Content {
        if text, ok := block.AsAny().(anthropic.TextBlock); ok {
            outputMessages = append(outputMessages, map[string]any{
                "role": "assistant", "content": text.Text,
            })
        }
    }

    safeOutput, err := guardMessages(ctx, aidrClient, outputMessages,
        aidr.AIGuardGuardChatCompletionsParamsEventTypeOutput)
    if err != nil || safeOutput == nil {
        fmt.Println("[AI Guard] Output blocked or guard error.")
        // Roll back the user turn we just appended
        conversation = conversation[:len(conversation)-len(safeInput)]
        continue
    }

    // Append the guarded assistant response to history
    conversation = append(conversation, safeOutput...)

    for _, m := range safeOutput {
        if msg, ok := m.(map[string]any); ok {
            fmt.Printf("Assistant: %v\n", msg["content"])
        }
    }
}
```

Key rules:
- Guard only the **new** message, not the entire history — avoids re-scanning and double-redacting already-processed content
- Store the **guarded (transformed) version** in history, not the original
- On output block, roll back the pending user turn to keep conversation history consistent

## Module Info

- Module: `github.com/crowdstrike/aidr-go`
- Go version: 1.24+
- Dependencies: `github.com/tidwall/gjson`, `github.com/tidwall/sjson`
- License: MIT
