# Concept: async, await, and the Runtime

> **Kind:** Concept. **This is the hard arc — read it twice.**

## The problem async solves

Your harness makes hundreds of API calls. Each one spends almost all its time *waiting* — for the network, for the model to generate. If you made them one at a time, synchronously, the program would sit idle during every wait. Async lets a single thread start a call, and while it waits, do other useful work (like starting the next call).

## What `async` and `await` actually mean

An `async fn` does not run when you call it. It returns a **future** — a value representing a computation that is not finished yet. The future does nothing until it is *driven*. You drive it with `.await`, which means "pause here until this future is ready, and let other work proceed meanwhile."

This is the mental shift: in synchronous code, calling a function runs it. In async code, calling an `async fn` gives you a *description* of work; `.await` is what actually advances it.

## The runtime

Futures need something to drive them — a scheduler that polls them, parks the ones that are waiting, and wakes them when their I/O is ready. That scheduler is the **async runtime**. Rust does not ship one in the standard library; the ecosystem standard is **`tokio`**. The `#[tokio::main]` attribute on `main` sets up the runtime so your top-level `async` code has something to run on.

## The lifecycle, precisely

It is worth having the exact lifecycle in mind rather than a vague sense of "it runs later." *Async Rust* describes a future's life this way: <cite index="0-1">when a future is created, it is idle — it has yet to be executed. Once executed, it can yield a value, resolve, or go to sleep because it is pending, and each subsequent poll returns either Pending or Ready until the future is resolved or cancelled.</cite> That polling loop — idle, then polled repeatedly, each poll answering "ready yet?" — is what the runtime is doing under every `.await`. You will rarely implement `poll` by hand, but knowing that `.await` is sugar over "poll this until it is `Ready`" demystifies the whole model.

The same book makes the concurrency payoff concrete with a kitchen analogy: several tasks each spending time waiting can overlap, because <cite index="0-6">the executor sets a task to idle when it hits an await and works on the next task in the queue while polling the idle ones.</cite> That single sentence is why your harness benefits from async at all — while one API call waits on the network, another can be in flight.

<div class="callout trap">
<span class="callout-label">The trap that stalls people</span>
Forgetting that an <code>async fn</code> does nothing until awaited. Calling <code>client.generate(prompt)</code> without <code>.await</code> produces a future and immediately discards the work — no request is made. The compiler warns about unused futures, which is one reason we deny <code>unused_must_use</code> in the workspace lints.
</div>

## Why this arc is genuinely harder

Async stacks several new things at once: futures, `await`, the runtime, and — next chapter — traits that contain async methods, plus the `Send + Sync` bounds that make futures safe to move between threads. Each is a real concept. Expect the compiler to be least forgiving here, and expect the errors to be longer.

The good news is that this arc rests directly on the ownership foundation from Part I. Most async borrow-checker fights are ownership and lifetime problems in disguise — a value moved into a future, a reference that does not live long enough — which is precisely the flows-and-moves model you already built. When an async error looks intimidating, the first move is to read it as an ownership question: *what owns this value, and how long does this borrow need to live?* The answer is usually there.

<div class="callout">
<span class="callout-label">Grounding</span>
The futures material in this arc follows the treatment in <em>Async Rust</em> (Flitton &amp; Morton, O'Reilly). Where a concept has a precise definition, we anchor to theirs rather than paraphrasing loosely, because "roughly right" on futures is how subtle async bugs get in.
</div>

## Questions to lock

1. What does an `async fn` return when you call it, and why does calling it *not* run the work?
2. What does `.await` do, and why does it let one thread stay busy during I/O waits?
3. What is the runtime's job, and why does `#[tokio::main]` exist?

{{#quiz ../../quizzes/03-async-concept-async.toml}}
