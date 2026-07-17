# Build: Family Spec + Validity Trait

> **Maps to:** Task 4. **Kind:** Build.

## Objective

Create the `panoptes-gen` crate. Define `FamilySpec` (the YAML shape, loaded with `serde_yaml`), the `ScenarioFamily` trait, and a first concrete family `CaGeo` whose `is_valid` encodes a real exclusion rule. Write the example `ca_geo.yaml`.

## Concepts exercised

- `serde_yaml` deserialization into a struct (same derive, different format).
- Defining and implementing a trait.
- Keeping content (YAML) and logic (Rust) on opposite sides of the boundary.

## The build loop (you drive)

1. **Write failing tests:** `FamilySpec::from_yaml` parses the fields; `CaGeo::is_valid` rejects the excluded combo and accepts valid ones.
2. **Predict:** the YAML parse test — what happens if a required field is missing from the YAML? Compile error or runtime `Err`? Why?
3. **Run**, check.
4. **Implement**.
5. **Run** green, **commit**.

## Done when

`cargo test -p panoptes-gen family` and `cargo test -p panoptes-gen validity` pass. You can articulate why the same `#[derive(Deserialize)]` works for YAML and JSON alike (serde is format-agnostic; the derive describes the *shape*, the format crate handles the *encoding*).
