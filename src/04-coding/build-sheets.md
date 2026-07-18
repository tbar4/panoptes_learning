# Build: Blinded Coding Sheets

> **Maps to:** Task 10. **Kind:** Build.

## Objective

Create the `panoptes-coding` crate. Write `make_sheet`, which splits response records into a `BlankSheetRow` (opaque id + response text + empty code fields) and a separate `KeyRow` (the id→model/params mapping coders never see). Add the seeded `shuffled_indices`. Prove the sheet leaks nothing.

## Scaffold

**Create** (new crate — add it to workspace `members`):

- `crates/panoptes-coding/Cargo.toml` — `[dependencies]`: `panoptes-core = { path = "../panoptes-core" }`, plus `serde`, `serde_json`, `thiserror`, `clap`, `anyhow` (all `{ workspace = true }`); `[dev-dependencies]`: `tempfile = "3"`. (`thiserror` is for the next chapter's error type — the original task plan omits it, so declare it now.)
- `crates/panoptes-coding/src/lib.rs` and `src/sheets.rs`.

**Expected result:** `cargo test -p panoptes-coding sheets` → **2 tests pass** (`sheet_does_not_leak_identity`, `shuffle_is_deterministic_and_a_permutation`).

## Concepts exercised

- Splitting one record into two types by what each audience may see.
- A seeded permutation without an external `rand` dependency (a small LCG).
- The blinding invariant as a test.

## The build loop (you drive)

1. **Write failing tests:** `sheet_does_not_leak_identity` (serialized sheet contains no model name, no vignette id, no parameters, but the key retains them); `shuffle_is_deterministic_and_a_permutation`.
2. **Predict:** the leak test serializes the sheet to JSON and asserts the string does not contain `"claude"`. What field on `BlankSheetRow` would make this test fail, and why is its absence the whole point?
3. **Run**, check.
4. **Implement**.
5. **Run** green, **commit**.

## Done when

`cargo test -p panoptes-coding sheets` passes. The blinding property is now guaranteed by the type, not by your memory.
