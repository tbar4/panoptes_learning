# Concept: Workspace Hygiene and the Language Boundary

> **Kind:** Concept.

## Two ideas in this short arc

**Workspace hygiene.** Lints are automated taste. Configuring `deny(unused_must_use)` turns the un-awaited-future mistake into a hard error across every crate. Setting `clippy::unwrap_used` to warn flags panics-in-waiting in library code (while leaving them fine in tests). Hygiene is what turns "code that compiles" into "code you would defend at a review."

**The language boundary.** This is the decision to keep Stages 5–6 in Python. It is not a failure of Rust ambition — it is choosing the right tool per stage. The statistics ecosystem for inter-rater reliability (Krippendorff's alpha, Cohen's kappa) is mature and correct in Python and thin in Rust. Reimplementing kappa by hand, in the crate whose entire purpose is a defensible instrument, would introduce exactly the kind of subtle correctness risk you are trying to eliminate. So the boundary is drawn at the file contract: Rust produces JSONL and CSV; Python reads them.

<div class="callout">
<span class="callout-label">The principle</span>
The interface is the file, not a language binding. That is what lets a typed Rust harness and a pandas analysis notebook coexist with zero friction — and it is the same fungibility argument from the data-model arc, applied to the whole pipeline. Draw boundaries at data, not at code.
</div>

## Questions to lock

1. What does a lint like `deny(unused_must_use)` buy you that code review alone does not?
2. Why is keeping reliability statistics in Python the *rigorous* choice, not the lazy one, for this specific instrument?
3. Why is "the interface is the file contract" the thing that makes the Rust/Python split painless?

{{#quiz ../../quizzes/05-hardening-concept-hygiene.toml}}
