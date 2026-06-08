# Decisions

This document records the changes made to bring the POWER steering documents to production quality. Each entry describes what changed, why it was wrong, and what failure mode it prevents in generated code.

---

## Response handling

### 1. Status check required before accessing `result`

**Files:** all three SDK docs, `POWER.md`

**Problem:** Examples accessed `response.result.blocked` (and nested fields) without first verifying `response.status == "Success"`. An `"Accepted"` (HTTP 202) response has a different `result` shape — `{ttl_mins, retry_counter, location}` — so accessing `.blocked` on it either throws an `AttributeError` / `TypeError` or silently returns a zero value.

**Change:** Every example that accesses `result` now checks status first:
- Python: `if response.status != "Success": raise RuntimeError(...)`
- TypeScript: `if (response.status !== "Success") throw new Error(...)`
- Go: `if resp.Status != "Success" { log.Fatalf(...) }`

A dedicated note in the Async / 202 Accepted section reinforces why the check is mandatory.

---

### 2. Nullability of `result`, `detectors`, and individual detector fields

**Files:** `python-sdk.md`, `typescript-sdk.md`, `POWER.md`

**Problem:** Examples accessed `response.result.blocked`, `response.result.detectors.malicious_prompt.detected`, etc., without guarding against `None` / `undefined`. The Pangea response schema makes `result` and every detector field optional — accessing them directly crashes on any response where the field is absent.

**Change:**
- Python: every example now checks `if not response.result:` before access; detector access uses `if detectors:` and `if detectors.malicious_prompt and detectors.malicious_prompt.detected`.
- TypeScript: `detectors` typed as `detectors?:` (optional); all accesses use optional chaining (`detectors?.malicious_prompt?.detected`).
- Go: explicitly documented as **value types** (not pointers) — zero-value safe, no nil check needed. This is noted in both the Response Type section and the Detectors note to prevent unnecessary nil guards being generated.

A nullability blockquote was added to the Response Structure section in `POWER.md` and to the Response Type section in each SDK doc.

---

### 3. Python `try/except` scope: SDK call only, not status check

**Files:** `python-sdk.md`

**Problem:** The Complete Example wrapped both the SDK call and the status check inside `try/except APIStatusError`. A `RuntimeError` raised by the status check propagated through the `except` clause, which only catches `APIStatusError`, so it was silently unhandled in the wrong scope.

**Change:** The `try/except APIStatusError` block now wraps only `client.guard_chat_completions(...)`. Status checks and all result access happen outside the try block, so control flow is unambiguous.

---

### 4. Go: `if err != nil` must terminate — no fall-through

**Files:** `go-sdk.md`

**Problem:** The error handling example lacked a `log.Fatal(err)` at the end of the `if err != nil` block. On a non-`*aidr.Error` network error, control fell through to the status check on a zero-value `resp`, which would then `log.Fatalf` with an empty status string, obscuring the real error.

**Change:** Added `log.Fatal(err)` as the final statement in the `if err != nil` block with a comment: `// always terminate — do not fall through to status check`.

---

### 5. Python async re-parsing: `model_validate()` not `**data`

**Files:** `python-sdk.md`

**Problem:** The `guard_with_polling` helper reconstructed the typed response model via `GuardChatCompletionsResponse(**data)`. Pydantic v2 models use field aliases and validators during construction; unpacking a raw dict with `**` bypasses them and may silently produce a partially-populated object or raise a validation error on aliased field names.

**Change:** Changed to `GuardChatCompletionsResponse.model_validate(data)`, which correctly applies Pydantic v2 validators and alias resolution.

---

## Type correctness

### 6. TypeScript: `event_type` must be a literal union, not `string`

**Files:** `typescript-sdk.md`

**Problem:** `event_type` was typed as `string` in the `ChatCompletionsGuard` interface. This provides no type safety — any string passes, including typos — and is inconsistent with Python (`Literal[...]`) and Go (typed constants).

**Change:** Changed to `"input" | "output" | "tool_input" | "tool_output" | "tool_listing"`.

---

### 7. TypeScript: `ChatMessage.role` must be a literal union

**Files:** `typescript-sdk.md`

**Problem:** Message objects constructed with `role: string` are not directly assignable to `Anthropic.MessageParam`, which requires `role: "user" | "assistant"`. Passing such messages to the Anthropic SDK requires a cast, which defeats type safety.

**Change:** Defined `type ChatMessage = { role: "user" | "assistant"; content: string }` and used `"assistant" as const` on constructed messages. The Guarding Model Output section notes explicitly that this narrow type is directly assignable to `MessageParam` without casting.

---

## Async / 202 handling

### 8. Async / 202 documented per-language with complete implementations

**Files:** all three SDK docs

**Problem:** The async sections were either absent or contained placeholder stubs that generated non-functional code.

**Change:** Each language now has a complete, working implementation:
- **TypeScript:** Documents the built-in `maxPollingAttempts` option (default: 5) and per-request override. The SDK polls automatically — no user code required.
- **Go:** Full `guardWithPolling` function using `time.NewTicker(5 * time.Second)` and `client.AIGuard.GetAsyncRequest` in a `select` loop that respects context cancellation.
- **Python:** Documents that the SDK has no first-class polling method. Provides a `guard_with_polling` helper that polls the `location` URL via `httpx.get` until `status == "Success"` or the `ttl_mins` deadline expires. Recommends increasing the client timeout as the primary mitigation.

---

## Missing parameters

### 9. `input_fpe_context` parameter added to all three SDK docs

**Files:** all three SDK docs

**Problem:** The `input_fpe_context` parameter — which decrypts FPE-redacted content before re-scanning in multi-turn conversations — was not documented in any language.

**Change:** Added to the parameter table in all three files with a description: "FPE context from a prior guard call; decrypts redacted content before scanning (multi-turn use)."

---

## Python-specific fixes

### 10. `ExtraInfoMcpTool` replaced with plain dicts

**Files:** `python-sdk.md`

**Problem:** The Tool Use Guarding section imported and used `ExtraInfoMcpTool` from `crowdstrike_aidr.models.ai_guard`. This class name was unverified and likely does not exist in the SDK — the import would raise `ImportError` at runtime.

**Change:** Replaced with plain Python dicts `{"server_name": ..., "tools": [...]}`, which is what the SDK's Pydantic model accepts via coercion. The `ExtraInfo` constructor accepts `mcp_tools` as a list of dicts.

---

### 11. `import json` missing from Tool Use Guarding section

**Files:** `python-sdk.md`

**Problem:** The tool input guarding example used `json.dumps(...)` without importing `json`.

**Change:** Added `import json` at the top of the Tool Use Guarding — Guarding Tool Inputs example.

---

### 12. Unredact `result` null guard added everywhere

**Files:** `python-sdk.md`

**Problem:** Several examples called `client.unredact(...)` and then accessed `unredacted.result.data` directly. `result` is Optional in Python — the same nullability rules that apply to guard responses apply to unredact responses.

**Change:** All unredact call sites now check `if not unredacted.result: raise RuntimeError("Empty result from unredact")` before accessing `.data`.

---

## Go-specific fixes

### 13. `toAnthropicMessages` undefined function removed

**Files:** `go-sdk.md`

**Problem:** The Multi-Turn Conversation Guarding example called `toAnthropicMessages(conversation)`, a helper function that was never defined anywhere in the document. Generated code would not compile.

**Change:** Replaced with an inline type-assertion loop that converts `[]any` to `[]anthropic.MessageParam` using `msg["role"].(string)` and `anthropic.NewUserMessage` / `anthropic.NewAssistantMessage`.

---

### 14. Environment variable naming documented

**Files:** `go-sdk.md`

**Problem:** The Go SDK natively reads `AIDR_BASE_URL_TEMPLATE` and `AIDR_API_TOKEN`, but examples in other languages used `AIDR_BASE_URL` and `AIDR_TOKEN`. Generated code using the "wrong" names in a multi-language deployment would silently fail to authenticate.

**Change:** The Environment Variables section now documents both naming schemes and explains the tradeoff: use the SDK-native names for Go-only deployments; use explicit `os.Getenv("AIDR_BASE_URL")` / `os.Getenv("AIDR_TOKEN")` reads for cross-language consistency.

---

### 15. `context.WithTimeout` added to Complete Example

**Files:** `go-sdk.md`

**Problem:** The Complete Example used `context.Background()` with no timeout. The Go SDK has no default timeout — without a deadline, a hung request blocks forever.

**Change:** Complete Example now opens with:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```
The Timeouts section documents this distinction from Python and TypeScript, which both default to 60 seconds.

---

### 16. `LlmProvider` / `Model` in `guardMessages` annotated as caller-configurable

**Files:** `go-sdk.md`

**Problem:** The `guardMessages` helper hardcoded `LlmProvider: "Anthropic"` and `Model: "claude-opus-4-6"`. Copied into a non-Anthropic context, these values would silently produce incorrect telemetry.

**Change:** Added inline comments: `// set to the provider generating the content being scanned` and `// set to the model actually generating the content being scanned`.

---

## TypeScript-specific fixes

### 17. `process.exit()` replaced with `throw new Error()`

**Files:** `typescript-sdk.md`

**Problem:** Blocked/error conditions in examples called `process.exit(0)` or `process.exit(1)`. This is inappropriate in library or server code — it terminates the entire process, bypassing any error handling in the caller.

**Change:** All such sites now `throw new Error(...)` with a descriptive message.

---

### 18. Unused imports and variables removed from Multi-Turn example

**Files:** `typescript-sdk.md`

**Problem:** The Multi-Turn example imported `* as readline` and created a `readline.Interface`, but the function signature took a plain `string` argument. The `Errors` namespace was also imported but never referenced. These would generate TypeScript compiler warnings or errors.

**Change:** Removed `import * as readline`, the `rl` variable, and the `Errors` import. Added commented construction lines for `client` and `anthropicClient` to show where they originate.

---

## Completeness

### 19. Tool Use Guarding sections added to all three SDK docs

**Files:** all three SDK docs

**Problem:** None of the SDK docs covered tool use guarding (`tool_input`, `tool_output`, `tool_listing` event types), despite these being core patterns for agentic applications.

**Change:** Added a Tool Use Guarding section to each SDK doc with three subsections: Guarding Tool Inputs (before execution), Guarding Tool Outputs (before feeding back to model), and Guarding Tool Listings (MCP). Each subsection includes a complete code example.

---

### 20. Multi-Turn Conversation Guarding sections added to all three SDK docs

**Files:** all three SDK docs

**Problem:** No SDK doc explained the multi-turn guarding pattern, including the critical rules about guarding only the new message (not full history), storing the guarded version, and rolling back the user turn on output block.

**Change:** Added a Multi-Turn Conversation Guarding section to each SDK doc with a complete working example and an explicit key rules list at the bottom.

---

### 21. Forward-reference blockquotes added after Complete Examples

**Files:** all three SDK docs

**Problem:** The Complete Example shows a single guard call in isolation. Without guidance, Kiro might treat this as the complete integration pattern rather than generating the full input-guard → LLM → output-guard pipeline.

**Change:** Added a blockquote after each Complete Example:
> **This example shows a single guard call in isolation.** For the full integration pattern — guarding input, calling an LLM, and guarding the output — see the **Guarding Model Output**, **Tool Use Guarding**, and **Multi-Turn Conversation Guarding** sections below.

---

## POWER.md fixes

### 22. Troubleshooting: 400 Bad Request guidance corrected

**File:** `POWER.md`

**Problem:** The 400 troubleshooting entry said "check messages array" — but `tool_listing` events use a `tools` key, not `messages`. Kiro following this advice for a `tool_listing` 400 would look in the wrong place.

**Change:** Updated to distinguish by event type:
> Check that `guard_input` contains the expected key: `messages` for `input`, `output`, `tool_input`, and `tool_output` events; `tools` for `tool_listing` events.

---

### 23. `McpTools` / `mcp_tools` removed from Complete Examples

**Files:** `go-sdk.md`, `typescript-sdk.md`

**Problem:** The Complete Examples for Go and TypeScript included `McpTools` / `mcp_tools` in a basic single-message `input` guard call. `mcp_tools` in `extra_info` is only meaningful for `tool_listing` events — including it in a basic input guard is misleading and implies it is required.

**Change:** Removed `mcp_tools` from the Complete Examples in both files. It remains only in the Tool Use Guarding → Guarding Tool Listings subsection where it is appropriate.
