# CrowdStrike AIDR SDK — Kiro POWER

This repository contains a [Kiro POWER](https://kiro.dev/docs/steerers/power/) for integrating the [CrowdStrike AIDR](https://www.crowdstrike.com/platform/ai-security/) SDK into LLM-powered applications. It provides steering documents that guide Kiro to generate correct, production-ready AI Guard integration code across Go, Python, and TypeScript.

## What is a POWER?

A POWER is a set of structured steering documents that Kiro (an AI coding assistant) reads to understand how to use a library or API. When you add this POWER to your Kiro workspace, Kiro can generate, review, and modify AIDR integration code with accurate knowledge of the SDK's behavior — including correct null handling, error patterns, multi-turn guarding, and async/polling flows.

## What does this POWER cover?

This POWER teaches Kiro how to integrate **CrowdStrike AI Guard** into applications that use an LLM (such as Claude, GPT, or Gemini). AI Guard scans prompts and responses for:

- Prompt injection and jailbreak attempts
- PII (SSNs, emails, phone numbers, etc.)
- Secrets and API keys
- Malicious URLs, domains, and IPs
- Custom entity patterns, topic classification, language detection, and competitor mentions

### Supported SDKs

| Language | Package | Import |
|----------|---------|--------|
| Python | `crowdstrike-aidr` (PyPI) | `from crowdstrike_aidr import AIGuard` |
| TypeScript | `@crowdstrike/aidr` (npm) | `import { AIGuard } from "@crowdstrike/aidr"` |
| Go | `github.com/crowdstrike/aidr-go` | `import "github.com/crowdstrike/aidr-go"` |

### Patterns covered

- Basic input/output guarding
- Multi-turn conversation guarding
- Tool use guarding (`tool_input`, `tool_output`, `tool_listing`)
- MCP tool listing with `extra_info.mcp_tools`
- FPE redaction and unredact workflows
- Async/202 polling
- Error handling and status checking

## Repository layout

```
POWER.md                  # Entry point — Kiro reads this first
steering/
  python-sdk.md           # Complete Python SDK reference
  typescript-sdk.md       # Complete TypeScript SDK reference
  go-sdk.md               # Complete Go SDK reference
```

## Using this POWER in Kiro

### Option 1: Reference directly

Add this repository as a POWER source in your Kiro workspace settings. Kiro will automatically read `POWER.md` and load the relevant language-specific steering file for your project.

### Option 2: Copy steering files

Copy `POWER.md` and the `steering/` directory into your project repository. Kiro will discover them automatically.

### Option 3: Selective import

Copy only the steering file for your language into your project's `steering/` directory and point Kiro at it.

## Quickstart

Once the POWER is active in your workspace, ask Kiro to:

- "Add CrowdStrike AI Guard to my LLM pipeline"
- "Guard all user inputs and model outputs with AI Guard"
- "Add multi-turn conversation guarding to this chat application"
- "Implement MCP tool listing protection with AI Guard"

Kiro will generate integration code that follows the patterns in the steering documents, including correct null checks, status validation, and async handling for your language.

## Language requirements

| Language | Minimum version |
|----------|----------------|
| Python | 3.12+ |
| Node.js / TypeScript | Node 18+ |
| Go | 1.24+ |

## Links

- [CrowdStrike AI Defense documentation](https://www.crowdstrike.com/platform/ai-security/)
- [Kiro POWER documentation](https://kiro.dev/docs/steerers/power/)
- [crowdstrike-aidr on PyPI](https://pypi.org/project/crowdstrike-aidr/)
- [@crowdstrike/aidr on npm](https://www.npmjs.com/package/@crowdstrike/aidr)
- [aidr-go on pkg.go.dev](https://pkg.go.dev/github.com/crowdstrike/aidr-go)

## Support

`@crowdstrike/aidr-aws-kiro-power` is a community-driven, open source project that integrates CrowdStrike AIDR security guardrails with your LLM Applications. While not a formal CrowdStrike product, it is maintained by CrowdStrike and supported in partnership with the open source developer community.

### Issue Reporting and Questions

Issues may be reported on [GitHub](https://github.com/CrowdStrike/aidr-aws-kiro-power/issues) and are used to track bugs, documentation updates, enhancement requests, and security concerns.

### Support Escalation

We endeavor to provide support for `@crowdstrike/aidr-aws-kiro-power` within the repository. This expands our online knowledge base, enables self-help for our community, and can reduce the time needed to receive answers.

If you are a CrowdStrike customer and would prefer to have your questions or issues handled directly with CrowdStrike Support, you are welcome to contact the CrowdStrike technical support team.
