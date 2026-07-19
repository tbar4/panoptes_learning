# Concept: Workspace Hygiene and the Language Boundary

> **Kind:** Concept.

## Two ideas in this short arc

**Workspace hygiene.** Lints are automated taste. Configuring `deny(unused_must_use)` turns the un-awaited-future mistake into a hard error across every crate. Setting `clippy::unwrap_used` to warn flags panics-in-waiting in library code (while leaving them fine in tests). Hygiene is what turns "code that compiles" into "code you would defend at a review."

Concretely, here is the mistake `unused_must_use` exists for — a fallible write whose `Result` is silently discarded:

```rust
fn append(line: &str) -> Result<(), String> {
    if line.is_empty() { return Err("refusing to log an empty line".into()); }
    Ok(())
}

fn main() {
    append("");   // the Result is silently discarded — did the write fail? unknowable
    println!("done");
}
```

By default that is merely a warning, easy to scroll past:

```text
warning: unused `Result` that must be used
 --> src/main.rs:7:5
  |
7 |     append("");
  |     ^^^^^^^^^^
```

For a harness whose append-only log *is* the dataset of record, a silently ignored write error is data loss. `unused_must_use = "deny"` promotes it to a compile error — the build fails until the `Result` is handled. The same mechanism is what catches a forgotten `.await`: an unused future is an unused must-use value, so "I called `generate` but no request ever happened" becomes unbuildable rather than a mystery.

Clippy's `unwrap_used` is the second guard, catching panics-in-waiting in library code:

```text
warning: used `unwrap()` on a `Result` value
  = note: if this value is an `Err`, it will panic
  = help: consider using `expect()` to provide a better panic message
```

The configuration is small — the spec block in the build chapter has the exact TOML — but the effect is workspace-wide: every crate inherits the same standards through `[lints] workspace = true`, so the rules are versioned with the code instead of living in someone's head.

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
