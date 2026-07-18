# Build: Family Spec + Validity Trait

> **Maps to:** Task 4. **Kind:** Build.

## Objective

Create the `panoptes-gen` crate. Define `FamilySpec` (the TOML shape, loaded with the `toml` crate), the `ScenarioFamily` trait, and a first concrete family `CaGeo` whose `is_valid` encodes a real exclusion rule. Write the example `ca_geo.toml`.

> **Why TOML for the spec:** it is the dialect you already speak (`Cargo.toml`), the `toml` crate is what Cargo itself builds on, and TOML has no implicit typing — a level named `"NO"` can never silently become `false` the way it can in YAML. For an instrument whose configs *are* the experimental conditions, unambiguous scalars beat terseness.

## Scaffold

**Create** (a whole new crate — remember to list it in the workspace `members`):

- `crates/panoptes-gen/Cargo.toml` — `[dependencies]`: `panoptes-core = { path = "../panoptes-core" }`, plus `serde`, `toml`, `tera`, `sha2`, `itertools`, `clap`, `anyhow` (all `{ workspace = true }`); `[dev-dependencies]`: `serde_json`.
- `crates/panoptes-gen/src/lib.rs`, `src/family.rs` (the `FamilySpec` + `from_toml`), `src/validity.rs` (the `ScenarioFamily` trait + `CaGeo`).
- `scenarios/families/ca_geo.toml` — the first family spec (content + metadata only).

**Dependencies this chapter exercises:** `toml` (spec parsing). `tera`/`sha2`/`itertools` sit unused until the next two chapters — declaring them now just saves manifest edits later.

**Expected result:** `cargo test -p panoptes-gen family` and `cargo test -p panoptes-gen validity` → **5 tests pass** between them.

## The spec (givens)

- `FamilySpec` fields: `name: String`, `version: u32`, `title: String`, `doctrine_refs: Vec<String>`, `action_menu: Vec<String>`, `template: String`. Derives: `Debug, Clone, Deserialize`. Constructor: `from_toml(s: &str) -> anyhow::Result<Self>`.
- The `ScenarioFamily` trait has exactly two methods: `fn name(&self) -> &str` and `fn is_valid(&self, p: &Params) -> bool`.
- `CaGeo`'s exclusion rule: a combination is invalid when `time_pressure` is `Hours` **and** `info_request` is `false` (no time to act and no way to gather data).
- The example `ca_geo.toml` content (family name, doctrine refs, action menu, template text) is data, not an exercise — copy it from the Answer Key.

Full code: [Answer Key, Task 4](../appendix-answer-key.html#task-4-family-spec-loading--validity-trait-stage-1a).

## Concepts exercised

- `toml` deserialization into a struct (same derive, different format).
- Defining and implementing a trait.
- Keeping content (TOML) and logic (Rust) on opposite sides of the boundary.

## The build loop (you drive)

1. **Write failing tests:** `FamilySpec::from_toml` parses the fields; `CaGeo::is_valid` rejects the excluded combo and accepts valid ones.
2. **Predict:** the TOML parse test — what happens if a required field is missing from the TOML? Compile error or runtime `Err`? Why?
3. **Run**, check.
4. **Implement**.
5. **Run** green, **commit**.

## Done when

`cargo test -p panoptes-gen family` and `cargo test -p panoptes-gen validity` pass. You can articulate why the same `#[derive(Deserialize)]` works for TOML and JSON alike (serde is format-agnostic; the derive describes the *shape*, the format crate handles the *encoding*).
