# Introduction

This is a course, not a manual. By the end you will have built **Panoptes** — a typed, reproducible evaluation harness in Rust — and, more importantly, you will understand *why* every piece is shaped the way it is. We are going to move the way a good pair-programming session moves: I frame a concept, you ask questions until it is solid, then we build the piece together and the compiler grades the work.

## What we are building

Panoptes is the execution engine behind an SDA decision benchmark. It does four things, and each maps to one arc of this course:

1. **Generate** parameterized scenario vignettes from templates and a parameter grid.
2. **Execute** those vignettes against version-pinned model APIs, logging every raw response.
3. **Code** the responses against a codebook, with human judgment recorded in types that reject invalid values.
4. **Validate** coder agreement (this last stage hands off to Python).

Stages 5 and 6 — statistical analysis and reporting — stay in Python and read the files this harness produces. We will not build them here; we will build the thing that produces their inputs.

## The end state, held in view

Everything we build serves one payoff you should keep in mind from the very first chapter: **a coder will be structurally unable to record a value that is not in the codebook.** Not "discouraged from," not "warned against" — unable, because the value will fail to parse. That property is what makes this instrument defensible, and it falls out almost for free once you understand Rust's type system and the `serde` library. When we reach it in Part II, it should feel inevitable rather than clever.

## How the arcs are sequenced

The order is chosen for *understanding*, not for delivery speed. Each arc teaches one core idea before the next arc depends on it:

- **Part I, Foundations** — the test loop, then ownership, borrowing, and moves. The mental model everything else assumes. New to Rust starts here.
- **Part II, Data Model** — `serde`, derive macros, enums-as-validation. Go slow here.
- **Part III, Generation** — traits, the config boundary, iterators. Modeling the problem in types.
- **Part IV, Async** — futures, `await`, `tokio`, mock-server testing. The genuinely hard arc; budget extra time.
- **Part V, Coding** — types for correctness properties you personally care about (blinding, parse-is-validation).
- **Part VI, Hardening** — workspace hygiene, lints, the language boundary.
- **Part VII, CLI** — unifying the binaries into one `panoptes` command with proper exit codes. Where it becomes a tool.

## A note on where you are starting from

You come to this with real strength in one area and newness in another, and it helps to be honest about both. You have built LLM-powered systems and agents, so the *domain* of this harness — models, prompts, evaluation, the shape of the problem — is familiar ground. What is new is **Rust itself**, and specifically its ownership model and its async story. That is exactly why this course front-loads concepts and adds a dedicated foundations arc before we write types in anger: we are not going to assume you already think in ownership and borrowing. We are going to build that mental model deliberately, because everything downstream — and especially the async arc — rests on it.

If a chapter feels hard, that is usually not a signal about your ability; it is the compiler teaching you a rule you have not yet internalized. The whole method is designed around making those lessons fast and legible rather than frustrating.

<div class="callout">
<span class="callout-label">The rhythm of each build chapter</span>
Before we run any code, I will ask you to <em>predict</em> what the compiler will say. The prediction is where the understanding forms. Getting it wrong is not a setback — a wrong prediction followed by the real error message is the single most efficient way to learn what the type system is actually enforcing.
</div>

Turn the page to see how the course is structured and how to run the build loop, then we start with Phase 0.
