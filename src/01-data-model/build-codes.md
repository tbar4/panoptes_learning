# Build: The Codebook Types

> **Maps to:** Task 2. **Kind:** Build.

## Objective

Create `codes.rs` in `panoptes-core`: the `ActionType` and `StrategicLogic` enums and the `CodedRow` struct. Write the test that is the whole reason we chose Rust — **an invalid codebook value must fail to deserialize.**

## Scaffold

**Create:** `crates/panoptes-core/src/codes.rs`. **Modify:** `crates/panoptes-core/src/lib.rs` (declare and re-export the module).

**Dependencies:** no manifest changes — `strum` (for `SCREAMING_SNAKE_CASE` serialization) and `serde_with` (for `NoneAsEmptyString` on `coder_notes`) are already in core's `Cargo.toml` from the previous chapter.

**Expected result:** `cargo test -p panoptes-core codes` → **3 tests pass** (`valid_logic_parses`, `invalid_logic_is_rejected`, `coded_row_json_roundtrip`).

## The spec (givens)

- `ActionType` variants: `SensorRetask, Maneuver, Monitor, EscalateToCommand, RequestData, NoAction`. `StrategicLogic` variants: `Control, Maritime, Political, Procedural, None, Mixed`.
- Both carry `#[strum(serialize_all = "SCREAMING_SNAKE_CASE")]` — wire form `"SENSOR_RETASK"`, `"CONTROL"` — and the same derive stack as the Part II enums.
- `CodedRow` fields, in order: `response_id: String`, `c1_action: ActionType`, `c2_logic: StrategicLogic`, `c2_anchor_quote: String`, `c3_escalation: u8`, `codebook_version: String`, `coder_notes: Option<String>` — the last annotated `#[serde_as(as = "NoneAsEmptyString")]` under a `#[serde_as]` struct attribute.

Full code: [Answer Key, Task 2](../appendix-answer-key.html#task-2-core-codebook-types-the-validation-by-type-payoff).

## Concepts exercised

- `#[strum(serialize_all = "SCREAMING_SNAKE_CASE")]` to match codebook string conventions.
- `serde_with::NoneAsEmptyString` for optional free-text fields.
- The negative test: asserting that a parse *fails*.

## The build loop (you drive)

1. **Write three failing tests:** a valid value parses; an invalid value (`"AGGRESSIVE"`) is rejected; a full `CodedRow` round-trips through JSON.
2. **Predict:** the `invalid_logic_is_rejected` test asserts `result.is_err()`. Before implementing, what makes it currently fail — the assertion, or the fact that the type does not compile yet?
3. **Run**, check.
4. **Implement** the enums and struct.
5. **Run** green, **commit**.

<div class="callout">
<span class="callout-label">The test that matters most</span>
<code>invalid_logic_is_rejected</code> is the load-bearing test of the entire harness. Everything about the instrument's integrity traces back to this one assertion passing. Write it deliberately.
</div>

<div class="callout">
<span class="callout-label">Best practice — why an enum, not a String</span>
<em>Effective Rust</em>'s Item 1 is "use the type system to express your data structures," and its guidance on exactly this choice is worth internalizing: prefer an enum over a wrapped primitive when <cite index="0-1">there's a chance that a new alternative could arise in the future.</cite> Codebook criteria are precisely that case — you may add a strategic-logic category during piloting — so an enum, not a string or a newtype-over-string, is the idiomatic and correct representation. The compiler then, in the book's phrase, "takes care of policing those semantics" for you.
</div>

## Done when

`cargo test -p panoptes-core codes` is green, and the invalid-value test proves the codebook constraint lives in the type rather than in a separate check.
