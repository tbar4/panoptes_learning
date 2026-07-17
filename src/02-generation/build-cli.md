# Build: The Generation CLI + Manifest

> **Maps to:** Task 6. **Kind:** Build.

## Objective

Add the `panoptes-gen` binary: a `clap` CLI that loads a family TOML, generates vignettes, writes each prompt to `prompts/`, and writes `manifest.csv` — the join spine every later stage reads. Test the manifest writer, then run the binary end to end.

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
