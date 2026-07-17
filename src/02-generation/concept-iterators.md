# Concept: Iterators and the Cartesian Product

> **Kind:** Concept.

## The idea

Generating vignettes means taking every combination of parameter values — 3 confidences × 2 time pressures × 2 reversibilities × 2 info-request states — and producing one vignette per valid combination. That "every combination" is a **cartesian product**, and Rust's iterator ecosystem expresses it cleanly.

`itertools::iproduct!` gives you the product as an iterator. You then `.filter()` out invalid combinations and `.map()` each surviving combination into a `Vignette`. The whole generation is a single iterator chain: product → filter → map → collect.

## Why iterators, not loops

You could write four nested `for` loops. The iterator chain is preferred here for reasons that matter beyond style: it is **lazy** (nothing computes until you `collect`), it is **composable** (each stage is independent and testable), and it makes the *shape* of the computation legible — "the cartesian product, minus invalid combos, each turned into a vignette" reads directly off the code.

<div class="callout">
<span class="callout-label">Rust iterators vs. Python generators</span>
If you know Python generators, iterators will feel familiar: both are lazy sequences. The difference is that Rust iterators are statically typed and monomorphized — the compiler often turns a chain into code as tight as the hand-written loop, so you pay nothing for the abstraction.
</div>

## Questions to lock

1. What is a cartesian product, and why is it the right description of "all parameter combinations"?
2. What does "lazy" mean for an iterator chain, and why is `collect()` the moment the work actually happens?
3. Why is a `filter` → `map` chain easier to test than the equivalent nested loops?

{{#quiz ../../quizzes/02-generation-concept-iterators.toml}}
