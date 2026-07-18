# Build: Vignette Generation

> **Maps to:** Task 5. **Kind:** Build.

## Objective

Write `generate.rs`: the `all_params()` iterator over the parameter grid, and `generate()` which filters by validity, renders each prompt through `tera`, hashes it, and produces `Vignette`s. Prove the counts and the uniqueness of IDs with tests.

## Scaffold

**Create:** `crates/panoptes-gen/src/generate.rs`. **Modify:** `crates/panoptes-gen/src/lib.rs` (re-export `all_params`, `generate`).

**Dependencies:** no manifest changes — `itertools` (`iproduct!`), `tera`, and `sha2` were declared when you created the crate.

**Expected result:** `cargo test -p panoptes-gen generate` → **4 tests pass** (`grid_has_24_raw_combinations`, `validity_filter_reduces_count` → 18, `ids_are_unique`, `template_renders_params`).

## The spec (givens)

- The confidence levels are `const CONFIDENCES: [u8; 3] = [30, 60, 95];` — a design decision from the study, not derivable.
- `all_params() -> impl Iterator<Item = Params>`: the cartesian product of confidences × both `TimePressure` variants × both `Reversibility` variants × `[true, false]` for `info_request` (3 × 2 × 2 × 2 = 24).
- `generate(family: &impl ScenarioFamily, spec: &FamilySpec) -> anyhow::Result<Vec<Vignette>>`.
- The tera context gets exactly four keys: `attribution_confidence` (number), `time_pressure` (its `Display` string), `info_request` (bool), `action_menu` (the vec joined with `", "`).
- `prompt_sha256` is the lowercase-hex SHA-256 of the rendered prompt bytes.

Full code: [Answer Key, Task 5](../appendix-answer-key.html#task-5-vignette-generation--manifest-stage-1b).

## Concepts exercised

- `itertools::iproduct!` over typed enum arrays.
- `tera` templating: injecting parameters into the prompt template.
- `sha2` hashing for the prompt fingerprint.

## The build loop (you drive)

1. **Write failing tests:** the raw grid has exactly 24 combinations; the validity filter reduces it to the expected number; all generated IDs are unique; a rendered prompt contains the injected parameter.
2. **Predict:** how many vignettes remain after `CaGeo`'s filter removes the `HOURS + NOINFO` combos? Work it out from the grid dimensions before running.
3. **Run**, check the arithmetic against reality.
4. **Implement**.
5. **Run** green, **commit**.

<div class="callout predict">
<span class="callout-label">Predict first</span>
The <code>ids_are_unique</code> test is a guard against a subtle bug: if two different parameter combinations produced the same ID, your manifest join would silently collapse them. Before running, convince yourself the ID construction cannot collide for distinct params.
</div>

## Done when

`cargo test -p panoptes-gen generate` passes, including the uniqueness guard.
