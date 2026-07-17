# Build: Workspace + Parameter Types

> **Maps to:** Task 1 in the task plan. **Kind:** Build (you write the code).

## Objective

Stand up the Cargo **workspace** and create `panoptes-core`, the crate that will hold every type crossing a boundary. Define the scenario parameter enums (`TimePressure`, `Reversibility`) and the `Params` struct, and prove with tests that they round-trip through strings and JSON.

## Concepts exercised

- Cargo **workspaces** and `[workspace.dependencies]` for shared version pinning.
- `#[derive(...)]` stacking: `Serialize, Deserialize, Debug, Clone, Copy, PartialEq`.
- `strum`'s `Display` + `EnumString` for enum↔string conversion.

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
