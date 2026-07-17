# Build: The Dispatch Loop + Run Binary

> **Maps to:** Task 9. **Kind:** Build.

## Objective

Write `dispatch.rs`: the opaque `response_id` function and the `dispatch` loop over vignettes × clients × epochs, writing one `ResponseRecord` per call. Then the `panoptes-run` binary that loads the manifest, builds the pinned client list, and runs the loop. Test the ID's opacity and the loop's record count against a mock.

## Concepts exercised

- Iterating over `&[Box<dyn ModelClient>]` and calling an async trait method.
- Deterministic-but-opaque ID construction (the blinding property, in the harness).
- An integration test that drives the whole loop against a mock server.

## The build loop (you drive)

1. **Write failing tests:** `response_id_is_deterministic_and_opaque` (same inputs → same id; epoch changes it; model name and params do NOT appear in it); `dispatch_writes_one_record_per_call` (2 vignettes × 1 client × 3 epochs → 6 records).
2. **Predict:** why must `response_id` be a *hash* rather than a readable concatenation? What property would a readable id break?
3. **Run**, check.
4. **Implement** dispatch and the binary.
5. **Run** green, **commit**.

<div class="callout">
<span class="callout-label">Milestone</span>
Stage 2 is complete. You can generate scenarios and run them against pinned models, logging every raw response to an append-only store, with response IDs that keep coders blind. Combined with Stage 1, you now have a working evaluation pipeline through execution — the parts that produce the dataset of record.
</div>

## Done when

`cargo test -p panoptes-harness` is fully green, including the opacity guard and the integration test.
