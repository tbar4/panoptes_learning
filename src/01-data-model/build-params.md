# Build: Workspace + Parameter Types

> **Maps to:** Task 1 in the task plan. **Kind:** Build (you write the code).

## Objective

Stand up the Cargo **workspace** and create `panoptes-core`, the crate that will hold every type crossing a boundary. Define the scenario parameter enums (`TimePressure`, `Reversibility`) and the `Params` struct, and prove with tests that they round-trip through strings and JSON.

## Scaffold

**Create** (this is a new project directory, separate from this book's repo):

- `Cargo.toml` — the workspace manifest. Declare **every shared dependency** for the whole course under `[workspace.dependencies]` now (full annotated list in the [Workspace Scaffold appendix](../appendix-scaffold.html)); later crates then just write `dep = { workspace = true }`.
- `crates/panoptes-core/Cargo.toml` — with `[dependencies]`: `serde`, `serde_with`, `strum`, `chrono` (all `{ workspace = true }`).
- `crates/panoptes-core/src/lib.rs` — module declarations + re-exports.
- `crates/panoptes-core/src/params.rs` — the types and their tests.

**Dependencies this chapter actually exercises:** `serde` (derives) and `strum` (`Display`, `EnumString`). Add `serde_json` under `[dev-dependencies]` for the JSON round-trip test.

**Expected result:** `cargo test -p panoptes-core params` → **2 tests pass** (`time_pressure_string_roundtrip`, `params_json_roundtrip`).

## Concepts exercised

- Cargo **workspaces** and `[workspace.dependencies]` for shared version pinning.
- `#[derive(...)]` stacking: `Serialize, Deserialize, Debug, Clone, Copy, PartialEq`.
- `strum`'s `Display` + `EnumString` for enum↔string conversion.

## What "round-trip" means, concretely

Encode a value, decode the result, and assert you end up with **exactly the value you started with** — while also pinning what the encoded form looks like in between. A round-trip test proves the codec is lossless and freezes the wire format, so a refactor that silently changes `"HOURS"` to `"Hours"` fails a test instead of corrupting every downstream join. These strings become the manifest CSV and the JSONL log — the file contract.

The two tests exercise **two independent codecs**, and they encode the same enum differently:

| Direction | Expression | Expected |
|---|---|---|
| enum → string (strum `Display`) | `TimePressure::Hours.to_string()` | `"HOURS"` |
| string → enum (strum `EnumString`) | `TimePressure::from_str("DAYS")` | `Ok(TimePressure::Days)` |
| rejection | `TimePressure::from_str("MINUTES")` | `Err(_)` |
| struct → JSON (serde) | `serde_json::to_string(&p)` | `{"attribution_confidence":60,"time_pressure":"Hours",…}` |
| JSON → struct (serde) | `serde_json::from_str::<Params>(&json)` | a `Params` for which `assert_eq!(p, back)` holds |

<div class="callout">
<span class="callout-label">Two codecs, not one</span>
<code>#[strum(serialize_all = "UPPERCASE")]</code> affects only <code>Display</code>/<code>FromStr</code> — so the string codec says <code>"HOURS"</code>. serde independently serializes the <em>variant name</em> — so JSON says <code>"Hours"</code>. That is fine: the manifest uses the strum codec in both directions, the JSONL log uses serde in both directions. But it is why each test must round-trip through its <em>own</em> codec, and why the enum→string direction is <code>Display</code>'s job — not <code>into()</code>/<code>try_into()</code>, which have no implementation here and will not compile.
</div>

## The build loop (you drive)

1. **Write the failing test** for `TimePressure` string round-tripping (`Hours` ↔ `"HOURS"`) and for `Params` JSON round-tripping. Derive them from the behavior described, not from the answer key.
2. **Predict** the failure — will it fail to *compile* or fail at *runtime*? (Hint: the type does not exist yet. What does the compiler say about that?)
3. **Run**, check the prediction.
4. **Implement** the enums and struct with the minimal derives to pass.
5. **Run** green, then **commit**.

<div class="callout predict">
<span class="callout-label">Predict first</span>
Before you write the implementation: which derive does the <em>string</em> round-trip need, and which does the <em>JSON</em> round-trip need? They are different traits from different crates. Naming them before you add them is the exercise.
</div>

## Done when

`cargo test -p panoptes-core params` shows two passing tests, and you can explain why `Copy` is safe to derive on these types (all fields are themselves `Copy`).

## Check yourself

Compare against Task 1 in the appendix task plan. If your derives differ, work out whether the difference matters — some are load-bearing (`Deserialize`), some are ergonomic (`Copy`).
