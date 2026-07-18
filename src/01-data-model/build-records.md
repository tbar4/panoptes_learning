# Build: Records, Vignettes, the File Contract

> **Maps to:** Task 3. **Kind:** Build.

## Objective

Create `vignette.rs` and `record.rs` in `panoptes-core`. Define `Vignette`, the deterministic `vignette_id` function, `Usage`, and `ResponseRecord` — the struct that becomes one line in `responses.jsonl`, the dataset of record. These types are the **file contract**: the interface every later stage and the Python analysis tail depend on.

## Scaffold

**Create:** `crates/panoptes-core/src/vignette.rs` and `crates/panoptes-core/src/record.rs`. **Modify:** `crates/panoptes-core/src/lib.rs` (re-export both).

**Dependencies:** no manifest changes — `chrono` (with its `serde` feature, for `DateTime<Utc>`) is already declared.

**Expected result:** `cargo test -p panoptes-core` → **7 tests pass** across the crate (2 params + 3 codes + 1 vignette + 1 record). The shared data model is complete.

## Concepts exercised

- `chrono::DateTime<Utc>` with serde support for timestamps.
- Deterministic ID construction from typed fields (no stringly-typed field access).
- Why the JSONL schema is the contract that makes the underlying tool fungible.

## The build loop (you drive)

1. **Write failing tests:** `vignette_id` produces the exact formatted string (`ca_geo-030-H-REV-INFO`); `ResponseRecord` round-trips through a JSON line.
2. **Predict** what `vignette_id` returns for a given `Params`, character by character, before running.
3. **Run**, check.
4. **Implement**.
5. **Run** green, **commit**.

## Done when

All of `cargo test -p panoptes-core` passes (params + codes + vignette + record). You now have the complete shared data model — every type the other three crates will import.

<div class="callout">
<span class="callout-label">Why this closes the arc</span>
Everything downstream reads or writes these types. Get them right and the generation, harness, and coding crates become mostly a matter of moving these values around. The data model is the hard thinking; the rest is plumbing.
</div>
