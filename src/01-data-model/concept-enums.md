# Concept: Enums as Validation

> **Kind:** Concept (read, do not code).

This is the payoff chapter the whole arc has been building toward. The idea is small and the consequence is large.

## The core idea

A Rust `enum` is a **closed set** of named alternatives. `StrategicLogic` is `Control` *or* `Maritime` *or* `Political` *or* `Procedural` *or* `None` *or* `Mixed` — and *nothing else can exist*. There is no seventh value. The compiler enforces this everywhere the type is used.

Now combine that with what you learned about `serde`: when `serde` **deserializes** a string into an enum, it matches the string against the known variants. A string that matches no variant is a **deserialization error**. Not a warning. Not a silently-accepted fallback. An error that stops the parse.

## Why this is the instrument's spine

In the Python sketch of this harness, validating a coded value meant a separate `lint.py` script that checked each value against an allowed list — a step you had to remember to run, that ran *after* the data already existed in a possibly-invalid state.

In Rust, the allowed list **is the type**. A coded row whose `c2_logic` field is `"AGGRESSIVE"` does not become an invalid `CodedRow` that you later catch — it *fails to become a `CodedRow` at all*. The validation is not a step in the pipeline; it is a property of the boundary. Parsing the file *is* validating it.

<div class="callout">
<span class="callout-label">The reframe</span>
"Parse, don't validate." Instead of accepting loose data and checking it afterward, you make the act of parsing reject anything malformed. The type system holds the invariant, so no downstream code can ever see a `CodedRow` with an off-codebook value — because such a value could not be constructed.
</div>

## Connecting back to the thesis

Recall why rubric validity is the primary intellectual risk: if the codebook reads as one person's opinion, the thesis fails at defense. The enum does not make the *criteria* defensible — that is the doctrinal grounding and the inter-rater reliability. But it does guarantee that the *recorded data* conforms exactly to the codebook you defined, with zero drift, which is one less thing a committee can poke. The instrument's categories are frozen in the type.

## Questions to lock

1. What is the difference between "validate the value after parsing" and "parse in a way that rejects invalid values"? Why does the second eliminate a class of bugs the first cannot?
2. If `StrategicLogic` has six variants, what happens — mechanically — when `serde` tries to deserialize `"CONTROL"`? What happens with `"AGGRESSIVE"`?
3. Why does putting the codebook's allowed values in an enum mean *no downstream code* ever needs to re-check them?

{{#quiz ../../quizzes/01-data-model-concept-enums.toml}}

Next: we turn `StrategicLogic` and friends into real code, and write the test that proves an invalid value is rejected.
