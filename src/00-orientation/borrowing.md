# Concept: Borrowing and References

> **Kind:** Concept. **Builds directly on the previous chapter — read that first.**

If every value has one owner and moving transfers it, how does a function *use* your value without stealing it? The answer is **borrowing**, and it is what the overwhelming majority of Rust function signatures actually do.

## Lending without giving up ownership

From *Rust for Rustaceans*: <cite index="0-5">Rust allows the owner of a value to lend out that value to others, without giving up ownership, through references. References are pointers that come with an additional contract for how they can be used.</cite>

A reference, written `&value`, lets code look at (or modify) a value it does not own. When the reference goes away, nothing is dropped — the reference never owned the value, so there is nothing to clean up. The owner keeps ownership the whole time. This is why you will see `&` everywhere: passing `&params` to a function lets it read your `Params` without moving it, so you can keep using `params` afterward.

Contrast this with the previous chapter, where passing an owned `String` *moved* it. Borrow instead, and the value stays yours:

```rust
fn describe(vignettes: &[String]) -> String {
    format!("{} vignettes ready", vignettes.len())
}

fn main() {
    let vignettes = vec!["ca_geo-030-H-REV-INFO".to_string(), "ca_geo-060-D-IRREV-NOINFO".to_string()];
    let msg = describe(&vignettes);          // lend it out...
    println!("{msg}");                        // 2 vignettes ready
    println!("still mine: {} entries", vignettes.len()); // ...and it is still ours
}
```

If `describe` took `vignettes: Vec<String>` instead, the second `println!` would be a compile error — value used after move. The `&` is the difference between lending and giving.

## Two kinds of reference, one iron rule

There are exactly two kinds, and the distinction is the whole game.

**Shared references (`&T`)** — read-only, and you can have as many as you like at once. The book: <cite index="0-6">a shared reference is a pointer that may be shared; any number of other references may exist to the same value, and values behind shared references are not mutable.</cite> Many readers, no writers.

```rust
fn main() {
    let prompt = String::from("You are the duty officer.");
    let a = &prompt;
    let b = &prompt;
    println!("{a} | {b} | {prompt}"); // three readers at once — no conflict
}
```

**Mutable references (`&mut T`)** — read-write, but **exclusive**. While a `&mut` exists, nothing else may touch the value. The book: <cite index="0-8">the compiler assumes that the mutable reference is exclusive</cite> — no other reference, shared or mutable, may coexist with it.

Put together, this is the rule the borrow checker enforces everywhere: **either any number of shared (`&`) references, or exactly one mutable (`&mut`) reference — never both at once.** Many readers *or* one writer. Not both.

<div class="callout">
<span class="callout-label">Why this rule buys so much</span>
This single constraint is how Rust guarantees memory safety <em>and</em> data-race freedom at compile time. A data race requires two things touching the same data at once with at least one writing — which is exactly the state the rule forbids. Get the "many readers or one writer" rule into your bones and most borrow-checker errors become predictable, because you will feel the violation before the compiler names it.
</div>

## Watch the rule fire

Here is the violation, in the smallest form you will actually meet — holding a reference into a `Vec` while pushing to it (a push may reallocate and move every element, which would leave `first` pointing at freed memory; the rule exists precisely to forbid this):

```rust,compile_fail
fn main() {
    let mut log = vec!["line 1".to_string()];
    let first = &log[0];          // shared borrow begins
    log.push("line 2".into());    // mutable borrow while shared is alive: ERROR
    println!("{first}");          // shared borrow still in use here
}
```

```text
error[E0502]: cannot borrow `log` as mutable because it is also borrowed as immutable
```

The fix is usually not `clone()` — it is *reordering*, so the shared borrow's flow ends before the mutable one begins:

```rust
fn main() {
    let mut log = vec!["line 1".to_string()];
    let first = &log[0];
    println!("{first}");          // last use — the shared borrow's flow ends here
    log.push("line 2".into());    // exclusive access is now fine
    println!("log has {} lines", log.len());
}
```

Same statements, different order, compiles clean. The compiler tracks flows by *last use*, not by scope braces — once `first` is used for the last time, its borrow is over.

## Back to the flows model

Remember flows from the last chapter — each value's line from creation to last use. Borrowing adds flows too: a shared borrow starts a flow that must not overlap a mutable one. The borrow checker's job, in the book's words, is to check that <cite index="0-6">there cannot be two parallel flows with mutable access to a value, nor a flow that borrows a value while there is no flow that owns the value.</cite> When you see "cannot borrow as mutable because it is also borrowed as immutable," that is two flows illegally overlapping. The fix is almost always to let one flow end (stop using the first reference) before the other begins.

## Where this lands in Panoptes

You do not need to *fight* the borrow checker much in this harness — the code is mostly straightforward — but you will read `&` and `&mut` constantly and need to know why each is there:

- `ModelClient::generate(&self, prompt: &str)` takes a **shared** reference to the client and a **shared** reference to the prompt string: it reads both, owns neither, so the caller keeps them.
- The append-only log writer opens a file and writes — the exclusivity of `&mut` on the file handle is what makes "one writer at a time" a compile-time fact.
- Passing `&vignettes` to the dispatch loop lets it read every vignette without moving the vector, so it is still yours afterward.

When an async error in Part IV mentions lifetimes or borrows, this is the model to reach for: *which flow is this, and does it overlap another?*

## Questions to lock

1. What does passing `&params` to a function let the function do, and what can you still do with `params` afterward?
2. State the borrow rule in one sentence. Why does forbidding "shared and mutable at once" prevent data races?
3. In the flows model, what illegal situation does the error "cannot borrow as mutable because it is also borrowed as immutable" describe?

{{#quiz ../../quizzes/00-orientation-borrowing.toml}}

That is the foundation. With ownership, moves, and borrowing solid, the rest of the course is mostly applying them. Part II begins — the data model, and the `serde` concept that makes the whole harness worth building in Rust.
