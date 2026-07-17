# Build: The Coded-CSV Loader

> **Maps to:** Task 11. **Kind:** Build.

## Objective

Write `validate.rs`: `load_coded`, which parses each coded line into a `CodedRow` — so an off-codebook value fails *here*, at the boundary — and `check_anchors`, enforcing the codebook rule that latent codes carry an anchor quote. Add the `panoptes-code` CLI. This is where the enums-as-validation payoff becomes a runnable command.

## Concepts exercised

- Parse-is-validation in practice: deserialization as the gate.
- Custom error types (`thiserror`) carrying line-number context.
- A CLI that exits non-zero on invalid data.

## The build loop (you drive)

1. **Write failing tests:** a valid row loads; a bad-enum row fails with the right line number; a missing anchor quote is caught by `check_anchors`.
2. **Predict:** the bad-enum test feeds `"c2_logic":"AGGRESSIVE"`. Where exactly does it fail — in `load_coded`'s `serde_json::from_str`, or in `check_anchors`? Trace the path before running.
3. **Run**, check.
4. **Implement**, then exercise the CLI on a deliberately bad file and confirm a non-zero exit.
5. **Run** green, **commit**.

<div class="callout">
<span class="callout-label">The payoff, realized</span>
This is the chapter the whole course pointed at. The codebook constraint you learned about abstractly in Part II is now a command that rejects invalid coded data at the boundary, with a line number, before it can pollute analysis. Parse <em>is</em> validation, running in a terminal.
</div>

## Done when

`cargo test -p panoptes-coding validate` passes and `panoptes-code --coded bad.jsonl` exits non-zero naming the offending line.
