# Contributing

## Core principle

**The steering documents are the source of truth.** The generated example files (`claude_example.py`, `claude_example.ts`, `claude_example.go`) are test artifacts — they exist to validate that the steering documents produce correct code when Kiro uses them. They are not committed to the repository and should never be edited directly to fix a bug. If generated code is wrong, fix the steering document.

## What lives where

| Path | Purpose |
|------|---------|
| `POWER.md` | Entry point. Kiro reads this first. Contains the overview, core concepts, quick-start snippets, and pointers to steering files. |
| `steering/python-sdk.md` | Complete Python reference. Everything Kiro needs to write correct Python AIDR integration code. |
| `steering/typescript-sdk.md` | Complete TypeScript reference. |
| `steering/go-sdk.md` | Complete Go reference. |

**Do not add** API documentation, architecture diagrams, changelog entries, or operational runbooks here. This repository contains only the steering content that Kiro reads.

## What belongs in a steering document

A steering document should contain everything a competent developer would need to write production-correct SDK integration code, and nothing else:

- Installation and import paths (exact package names and import statements)
- Client construction with all relevant options
- Method signatures and parameter tables
- Return types with nullability/optionality clearly marked
- Error types, hierarchy, and handling patterns
- Response status semantics (`"Success"` vs `"Accepted"`)
- Async/polling behavior
- Complete, runnable examples for each major pattern
- Explicit gotchas that differ from obvious or analogous APIs

**Do not include** marketing copy, conceptual background that is already in `POWER.md`, or information that can be directly inferred from the method signature.

## How to test a change

Steering documents are tested by running Kiro against them and inspecting the generated output.

1. Make your change to the relevant `steering/*.md` or `POWER.md`.
2. Open a Kiro session in a scratch directory with the relevant language toolchain available.
3. Ask Kiro to implement a representative integration:
   - "Add CrowdStrike AI Guard input and output guarding to this application"
   - "Add multi-turn conversation guarding with AI Guard"
   - "Implement MCP tool listing protection"
4. Review the generated code for correctness:
   - Does it check `response.status == "Success"` before accessing `result`?
   - Does it guard `result` / `result.detectors` for nullability (Python/TypeScript)?
   - Does it handle the blocked and transformed cases?
   - Are imports correct and complete?
   - Does it compile / pass type-checking?
5. If the generated code is wrong, revise the steering document and repeat.

You can also run the SDK directly against a live AI Guard endpoint to confirm runtime behavior.

## Editing guidelines

### Examples must be complete and correct

Every code example must be copy-paste runnable (modulo credentials). No undefined functions, no missing imports, no placeholder logic that silently swallows errors.

### Nullability must be explicit

Python and TypeScript: `result`, `result.detectors`, and each detector field are `Optional` / `?`. Every example that accesses these must show the null check.

Go: `Result` and `Detectors` are value types — no nil check needed. This must be noted explicitly so Kiro does not generate unnecessary nil guards.

### Status check before result access

Every example that accesses `result` must first check `response.status == "Success"` (Python), `resp.Status != "Success"` (Go), or `response.status === "Success"` (TypeScript). No exceptions. Accessing `result` on an `"Accepted"` response crashes because the shape is different.

### Error handling must not fall through

In Go, the `if err != nil` block must always terminate (return or log.Fatal). Do not let control fall through to a status check on a zero-value response.

In Python, the `try/except APIStatusError` block should wrap only the SDK call. Status checks and result access belong outside the try block.

### One example per pattern

Each major pattern (guarding input, guarding output, tool use, multi-turn, unredact) gets its own section with a self-contained example. Don't combine unrelated patterns into a single code block.

### Cross-language consistency

When the same concept exists in all three languages, the explanation and pattern should be consistent. Differences in behavior (e.g., Go value types vs Python Optional, TypeScript auto-polling vs Python manual polling) should be documented explicitly in each language's file, not just in one.

## Adding a new section

1. Identify the pattern. What would a developer need to implement? What are the failure modes?
2. Write the section in all three language files (if applicable).
3. Add a pointer in `POWER.md` if the concept is language-agnostic (e.g., a new event type or a new detector).
4. Test with Kiro (see above).

## Updating for SDK version changes

When a new SDK version changes behavior:

1. Update the affected `steering/*.md` file(s).
2. Update package version references if the new behavior requires a minimum version.
3. Update `POWER.md` if the core concept changed.
4. Re-test with Kiro to confirm generated code is correct on the new version.

## Pull request checklist

- [ ] Steering document change, not a generated example file
- [ ] All code examples compile/parse cleanly
- [ ] Null checks present wherever Python/TypeScript optionality applies
- [ ] Status check present before every `result` access
- [ ] Go `if err != nil` blocks terminate without fall-through
- [ ] Imports are complete in every example
- [ ] Tested with Kiro: generated output matches expected pattern
- [ ] `POWER.md` updated if a new concept was added
