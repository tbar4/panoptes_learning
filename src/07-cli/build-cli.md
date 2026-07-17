# Build: The Unified `panoptes` Command

> **Maps to:** Task 13 (new — extends the plan). **Kind:** Build.

## Objective

Create a top-level `panoptes` binary crate whose CLI is a `clap` subcommand enum wrapping the three stages: `generate`, `run`, and `code`. Make `main` return a type that produces the right OS exit code, and write an **integration test** (with `assert_cmd`) that runs the whole pipeline — generate, then run against a mock, then validate — and asserts the exit codes.

## Concepts exercised

- `clap` derive with a `#[command(subcommand)]` enum.
- `main` returning `Result` / `ExitCode` for honest exit status.
- Integration tests in `tests/` that invoke the built binary with `assert_cmd`.
- Composability: proving a failed stage produces a non-zero exit.

## File structure

```
crates/panoptes-cli/
├── Cargo.toml            # depends on gen, harness, coding
└── src/
    └── main.rs           # the subcommand enum + dispatch
tests/
└── pipeline.rs           # end-to-end integration test
```

The three existing binaries can stay (they are handy for focused runs), or become thin shims. The new `panoptes` command is the front door.

## The build loop (you drive)

1. **Write the failing integration test** first, in `tests/pipeline.rs`:
   - `panoptes generate --family ...` exits 0 and produces a manifest.
   - `panoptes code --coded <a-known-bad-file>` exits **non-zero** (the composability guarantee).
   - Optionally, the full happy-path chain runs green.
2. **Predict:** `assert_cmd` runs the compiled binary as a subprocess. Before running, what does the test assert about the *bad* case — a specific exit code, or merely "failure"? Decide what "failure" should mean for `panoptes code`.
3. **Run**, check — it fails because the `panoptes` binary does not exist yet.
4. **Implement** the subcommand enum and dispatch, wiring each variant to the stage function you already built. Return `ExitCode` so an invalid-data error becomes a non-zero exit.
5. **Run** green, **commit**.

<div class="callout predict">
<span class="callout-label">Predict first</span>
The subcommand enum is a closed set — <code>Generate</code>, <code>Run</code>, <code>Code</code>. When you <code>match</code> on it, the compiler checks you handled every variant. Before you write the match: what happens if you later add a <code>Report</code> subcommand and forget to handle it? (This is exhaustiveness — the same safety net enums gave you for codebook values, now guarding your CLI dispatch.)
</div>

## Done when

`cargo test` runs the integration test green, `panoptes --help` lists the three subcommands, and `panoptes code --coded bad.csv` exits non-zero — provably composable in a `&&` chain or a CI step.

<div class="callout">
<span class="callout-label">Milestone — it is a tool</span>
The harness is now a single command with discoverable verbs and honest exit codes. This is the artifact you would hand to a collaborator on the TAP Lab engagement or demo at a defense: not three scripts, but one typed, tested <code>panoptes</code> tool whose validation failures stop a pipeline. Every correctness property built across the course is now reachable from one front door.
</div>
