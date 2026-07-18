# Build: ModelClient + the Append-Only Log

> **Maps to:** Task 7. **Kind:** Build.

## Objective

Create the `panoptes-harness` crate. Define the `ModelClient` trait (with `#[async_trait]`) and `ModelResponse`. Write the append-only JSONL writer and the test that guards its most important property: **a second append must not overwrite the first.**

## Scaffold

**Create** (new crate — add it to workspace `members`):

- `crates/panoptes-harness/Cargo.toml` — `[dependencies]`: `panoptes-core = { path = "../panoptes-core" }`, plus `serde`, `serde_json`, `reqwest`, `tokio`, `async-trait`, `chrono`, `sha2`, `clap`, `anyhow` (all `{ workspace = true }`); `[dev-dependencies]`: `wiremock = "0.6"`, `tempfile = "3"`. (`sha2` is for the dispatch chapter's response-id hashing — the original task plan omits it there, so declare it now.)
- `crates/panoptes-harness/src/lib.rs`, `src/client.rs` (the trait), `src/jsonl.rs` (the writer).

**Dependencies this chapter exercises:** `async-trait`, `serde_json` (one record per line), `tempfile` (dev).

**Expected result:** `cargo test -p panoptes-harness jsonl` → **1 test passes** (`appends_without_truncating`).

## Concepts exercised

- Defining an async trait with `async_trait`.
- `std::fs::OpenOptions` with `.append(true)` for append-only semantics.
- Testing filesystem behavior with `tempfile`.

## The build loop (you drive)

1. **Write the failing test** `appends_without_truncating`: write two records, assert the file has two lines and each is valid JSON.
2. **Predict:** what would the test show if you opened the file with `.create(true).write(true)` instead of `.append(true)`? (This is the bug the test exists to catch.)
3. **Run**, check.
4. **Implement** the writer.
5. **Run** green, **commit**.

<div class="callout">
<span class="callout-label">Why this invariant is sacred</span>
<code>responses.jsonl</code> is the dataset of record — the thing the repo archives and a replicator re-analyzes. If a run could truncate it, a single mistake destroys evidence. The append-only test encodes that the log only ever grows.
</div>

## Done when

`cargo test -p panoptes-harness jsonl` passes and the append-only property is proven.
