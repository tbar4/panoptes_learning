# Build: The Anthropic Client

> **Maps to:** Task 8. **Kind:** Build.

## Objective

Implement `AnthropicClient` — a concrete `ModelClient` that builds the request, sends it, and parses the completion into a `ModelResponse`. Test it entirely against a `wiremock` mock server, spending no API credits.

## Scaffold

**Create:** `crates/panoptes-harness/src/anthropic.rs`. **Modify:** `crates/panoptes-harness/src/lib.rs` (re-export `AnthropicClient`).

**Dependencies:** no manifest changes — `reqwest` and the dev-only `wiremock` were declared when you created the crate. The test needs `#[tokio::test]`, which `tokio`'s `full` features already cover.

**Expected result:** `cargo test -p panoptes-harness anthropic` → **1 test passes** (`parses_a_mocked_completion`).

## Concepts exercised

- `reqwest` JSON POST with headers.
- Deserializing a nested API response into typed structs.
- `wiremock` mock definitions and `#[tokio::test]`.

## The build loop (you drive)

1. **Write the failing test** `parses_a_mocked_completion`: mount a mock returning a known completion + usage, call `generate`, assert the parsed text and token counts.
2. **Predict:** the response JSON has a `content` array of blocks. What does your parsing do if the array is empty? Decide the behavior before implementing.
3. **Run**, check.
4. **Implement** the client with an injectable `base_url`.
5. **Run** green, **commit**.

<div class="callout trap">
<span class="callout-label">Verify against live docs</span>
The request and response shapes here are written to match the API's structure, but API schemas change. When you build this for real, check the current request/response format against the official documentation before trusting the parse. The mock proves your parsing logic; it does not prove the schema is current.
</div>

## Done when

`cargo test -p panoptes-harness anthropic` passes. Adding a second provider (OpenAI, a local gateway) is now "copy this file, change the request/response shapes" — the trait makes providers uniform.
