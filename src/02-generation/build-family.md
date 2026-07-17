# Build: Family Spec + Validity Trait

> **Maps to:** Task 4. **Kind:** Build.

## Objective

Create the `panoptes-gen` crate. Define `FamilySpec` (the TOML shape, loaded with the `toml` crate), the `ScenarioFamily` trait, and a first concrete family `CaGeo` whose `is_valid` encodes a real exclusion rule. Write the example `ca_geo.toml`.

> **Why TOML for the spec:** it is the dialect you already speak (`Cargo.toml`), the `toml` crate is what Cargo itself builds on, and TOML has no implicit typing — a level named `"NO"` can never silently become `false` the way it can in YAML. For an instrument whose configs *are* the experimental conditions, unambiguous scalars beat terseness.

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
