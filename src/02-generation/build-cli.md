# Build: The Generation CLI + Manifest

> **Maps to:** Task 6. **Kind:** Build.

## Objective

Add the `panoptes-gen` binary: a `clap` CLI that loads a family TOML, generates vignettes, writes each prompt to `prompts/`, and writes `manifest.csv` — the join spine every later stage reads. Test the manifest writer, then run the binary end to end.

## Scaffold

**Create:** `crates/panoptes-gen/src/main.rs`. **Modify:** `crates/panoptes-gen/Cargo.toml` — add a `[[bin]] name = "panoptes-gen" path = "src/main.rs"` section and `tempfile = "3"` under `[dev-dependencies]` (the manifest test writes to a temp dir).

**Expected result:** `cargo test -p panoptes-gen manifest` → **1 test passes**; then `cargo run -p panoptes-gen -- --family scenarios/families/ca_geo.toml` prints `generated 18 vignettes → scenarios/generated` and leaves `manifest.csv` plus 18 files under `scenarios/generated/prompts/`.

## The spec (givens)

- CLI args: `--family <PathBuf>` (required) and `--out <PathBuf>` defaulting to `"scenarios/generated"`.
- The manifest header is exactly `vignette_id,family,attribution_confidence,time_pressure,reversibility,info_request,prompt_sha256`, then one row per vignette using each enum's `Display` string. No prompt body in the manifest.
- Each prompt is written to `<out>/prompts/<vignette_id>.txt`; the closing message goes to **stderr**: `generated {n} vignettes → {out}`.

Full code: [Answer Key, Task 6](../appendix-answer-key.html#task-6-generation-cli--manifest-writer-stage-1c).

## Concepts exercised

- `clap` derive for command-line argument parsing.
- Filesystem writing and directory creation.
- The manifest as the single place parameters live for the join.

## The build loop (you drive)

1. **Write a failing test** for `write_manifest`: header present, one row per vignette, IDs appear.
2. **Predict** the line count of the manifest for N vignettes (remember the header).
3. **Run**, check.
4. **Implement**, then run the actual binary: `cargo run -p panoptes-gen -- --family scenarios/families/ca_geo.toml`.
5. **Verify** the output files exist, **commit**.

## Done when

The binary emits `manifest.csv` plus one prompt file per vignette, and you can point to the manifest as the reason Stage 5 analysis will be a simple join rather than forensic reconstruction.

<div class="callout">
<span class="callout-label">Milestone</span>
Stage 1 is complete. You can generate a reproducible, parameterized scenario set from a template and a grid, with a manifest that ties every vignette to its parameters. That is a real, runnable piece of the harness.
</div>
