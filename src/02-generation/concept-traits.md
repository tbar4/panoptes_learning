# Concept: Traits and the Config Boundary

> **Kind:** Concept.

## Traits, briefly

A **trait** is a set of behaviors a type promises to provide — Rust's version of an interface. `ScenarioFamily` will require any family type to answer `name()` and `is_valid(&Params)`. Code can then work with *any* family through the trait, without knowing which concrete family it is.

You have used traits already (`Serialize`, `Display` are traits). Here you *define* one for the first time in this project, which is a small but real step up.

## The design decision that matters

The consequential idea in this chapter is not the trait mechanics — it is **where the validity logic lives.** In the Python sketch, families carried a list of string "valid_rules" that were evaluated at runtime. That means: logic expressed as data, interpreted while the program runs, with errors surfacing only when a bad rule is hit.

We are making the opposite choice. **YAML holds content and metadata only** — the prompt template, the doctrine references, the action menu. The logic of *which parameter combinations are meaningful* lives in Rust code, in each family's `is_valid` implementation. That means the compiler checks it, and a malformed rule is a compile error, not a runtime surprise.

<div class="callout trap">
<span class="callout-label">The trap this avoids</span>
Putting logic in config feels flexible — "just edit the YAML, no recompile." But config-as-logic is untyped, unchecked, and evaluated late. For a reproducible instrument, you want the rules that shape your scenario set to be as scrutinized as any other code. Flexibility you cannot verify is a liability here, not a feature.
</div>

## Questions to lock

1. What does a trait let you do that you could not do by writing a function per concrete type?
2. Why is "which parameter combinations are valid" logic, and therefore code, rather than data that belongs in YAML?
3. What do you lose by moving that logic out of config, and why is the trade worth it for this project specifically?

{{#quiz ../../quizzes/02-generation-concept-traits.toml}}
