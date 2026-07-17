# Concept: Async Traits, Send + Sync

> **Kind:** Concept.

## Why this needs its own chapter

You want a `ModelClient` trait with an `async fn generate`. But plain Rust traits could not, for a long time, contain async methods directly, and even now the ergonomic path for a trait used as `Box<dyn ModelClient>` is the **`async_trait`** macro. It rewrites your async trait methods into a form that works with dynamic dispatch. You annotate the trait and its impls with `#[async_trait]` and write async methods as if it just worked.

## Send + Sync, briefly

Because the runtime may move futures between threads, the things inside them must be safe to send across threads. Two marker traits express this: **`Send`** (safe to move to another thread) and **`Sync`** (safe to share by reference across threads). When you write `Box<dyn ModelClient>`, you will often need `ModelClient: Send + Sync` so the boxed clients can be used by the multi-threaded runtime.

You will not usually *implement* these — they are automatic for most types. But you will *require* them in bounds, and the compiler will tell you when a bound is missing, sometimes with an error that points several layers away from the real cause. Learning to read those is part of this arc.

<div class="callout">
<span class="callout-label">Dynamic dispatch recap</span>
<code>Box&lt;dyn ModelClient&gt;</code> means "a heap-allocated something that implements ModelClient, decided at runtime." It lets you hold a list of different client types (Anthropic, OpenAI, a local model) in one <code>Vec</code> and call <code>generate</code> on each uniformly. The cost is a pointer indirection; the benefit is the uniform dispatch loop.
</div>

## Questions to lock

1. Why do we reach for the `async_trait` macro instead of just writing `async fn` in the trait?
2. What do `Send` and `Sync` guarantee, and why does the multi-threaded runtime need them for boxed clients?
3. What does `Box<dyn ModelClient>` buy us that a concrete type would not, for the dispatch loop?

{{#quiz ../../quizzes/03-async-concept-traits-async.toml}}
