# Build: Lints, Handoff Contract, README

> **Maps to:** Task 12. **Kind:** Build.

## Objective

Add workspace-wide lints, write the Stage-4 reliability handoff contract (a README documenting what Python reads and produces — not implemented in Rust), and write the workspace README with the run sequence and the invariants. Then a full `cargo build --workspace && cargo test --workspace && cargo clippy --workspace`.

## Scaffold

**Create:** `reliability/README.md` (the Python handoff contract) and the workspace `README.md`. **Modify:** the root `Cargo.toml` (add `[workspace.lints.rust]` / `[workspace.lints.clippy]`) and **every** crate's `Cargo.toml` (add `[lints] workspace = true` below `[package]`).

**Dependencies:** none — this chapter adds configuration and documentation, not code.

**Expected result:** `cargo build --workspace && cargo test --workspace && cargo clippy --workspace` all clean (26 tests across the four crates: 7 core + 10 gen + 4 harness + 5 coding), with clippy warning only on intentional library `unwrap`s, if any.

## Concepts exercised

- `[workspace.lints]` and per-crate `[lints] workspace = true`.
- Documenting a language boundary as an explicit contract, not an afterthought.
- A full-workspace green build as the definition of done.

## The build loop (you drive)

1. Add the workspace lints; wire each crate to inherit them.
2. Write `reliability/README.md`: the exact files Python reads (`primary.csv`, `second_coder.csv`, `blind_key.csv`), what it produces (`irr_report.csv`, disagreement dumps), and why Python.
3. Write the workspace `README.md`: the run sequence and the four invariants (append-only log, blinded sheets, enums-as-validation, manifest-as-join-spine).
4. Run the full-workspace build, test, and clippy. Fix any clippy warnings in library code.
5. **Commit.**

<div class="callout">
<span class="callout-label">Milestone — the harness is done</span>
Stages 1–4 exist, typed and tested, with the Python handoff specified as a contract. This is no longer a set of exercises; it is software you could hand to a committee or a client and defend line by line. The thesis harness and the contract harness are the same artifact.
</div>

## Done when

`cargo build --workspace && cargo test --workspace && cargo clippy --workspace` is clean, and the two READMEs make the pipeline runnable and the language boundary explicit.
