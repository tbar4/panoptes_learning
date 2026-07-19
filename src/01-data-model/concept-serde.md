# Concept: serde and the derive Macro

This is the data-model arc, and everything in it exists to earn one payoff. This chapter builds the foundation for that payoff: understanding what `serde` is, what a `derive` macro actually does, and why generating conversion code at compile time changes when your bugs surface.

## The problem serde solves

The harness lives or dies on moving structured data across boundaries. A scenario becomes a prompt file. A model response becomes a line in a log. A coder's judgment becomes a row in a sheet. Every one of those is the same underlying operation: a Rust value in memory turning into text on disk, or text on disk turning back into a Rust value.

In Python you would reach for `json.dumps` and `json.loads` and mostly not think about it, because Python does not care what shape the thing is until it breaks at runtime. If a field is missing or the wrong type, you find out when the code hits that line — often in production, often at the worst time.

**serde** is Rust's answer. The name is just "**ser**ialize / **de**serialize." It is a library, not a language feature, but it is so universal it might as well be built in.

## The part that looks like magic

Here is the thing that feels like magic at first, and that is worth demystifying now so it does not stay magic: **you almost never write the conversion code yourself.** You annotate a struct like this —

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct Params {
    attribution_confidence: u8,
    info_request: bool,
}

fn main() {
    let p = Params { attribution_confidence: 60, info_request: true };

    let json = serde_json::to_string(&p).unwrap();
    println!("{json}"); // {"attribution_confidence":60,"info_request":true}

    let back: Params = serde_json::from_str(&json).unwrap();
    assert_eq!(p, back);
    println!("round-tripped: {back:?}");
}
```

— and `serde` generates the conversion for you, tailored to that exact type. Run it: field names become JSON keys, types map to JSON values, and the trip back reproduces an equal struct — without you writing a line of conversion code. The code that turns a `Params` into JSON is *written, by the macro, before your program ever starts.* There is no reflection, no runtime inspection, no dictionary of fields being walked while your program runs, the way Python does it.

## What "derive" actually is

This is the piece most people treat as incantation. Let us not.

A **derive macro** is a code generator that runs during compilation. When you write `#[derive(Serialize)]` above a struct, you are telling the compiler: *look at the fields of this struct, and write the serialization function for me.* The macro reads your struct's shape — its field names and their types — and emits real Rust source code: an implementation of the `Serialize` trait. That generated code then gets compiled right alongside everything you wrote by hand.

You have already met this mechanism without thinking about it. `#[derive(Debug)]`, which lets you print a value with `{:?}` for inspection, is the same thing: it generates the code that formats your type for debugging. `#[derive(Clone)]` generates the code to duplicate it. `serde`'s macros are more elaborate, but mechanically identical: read the type, emit the code.

## Why this matters: bugs move to compile time

Here is the consequence that makes `serde` the right tool for a reproducibility-critical instrument. Because the generation happens **at compile time against your specific type**, the compiler can see the whole thing. If a field's type cannot be serialized, you find out when you build — not when a log write fails at two in the morning during your evaluation run. Watch it happen:

```rust,compile_fail
use serde::Serialize;

#[derive(Serialize)]
struct Bad {
    name: String,
    handle: std::fs::File, // what would serializing an open file even mean?
}

fn main() {}
```

```text
error[E0277]: the trait bound `File: serde::Serialize` is not satisfied
```

The program never existed. In Python, the equivalent (`json.dumps` on an object holding a file handle) is a `TypeError` — at runtime, on the code path that happened to hit it.

This is the first appearance of a theme that runs through the entire course:

<div class="callout">
<span class="callout-label">The recurring theme</span>
<strong>Errors move from runtime to compile time.</strong> Python catches a malformed record when it tries to use it. Rust, with serde, catches an unserializable type when you compile. For an instrument whose credibility depends on the data being exactly what you claim it is, moving that whole class of failure to build time is not a convenience — it is the argument for the language.
</div>

## Why this arc is the foundation

Hold the end state in view, because it is *why* we start here. In a few chapters, your codebook criteria become Rust enums — `StrategicLogic` with variants `Control`, `Maritime`, `Political`, and so on. The reason that works — the reason a coder literally cannot record a value outside the codebook — is that `serde`'s deserialization of an enum will **reject any string that does not match a variant.**

That rejection is generated by the same derive mechanism we just discussed, applied to an enum instead of a struct. So the "parse is validation" property that makes this whole harness worth building in Rust is not a separate trick you will learn later. It is *this concept* — `serde` derive — pointed at an enum. Get this arc solid and that payoff is almost free. Rush it and the payoff will feel like luck.

## Questions to lock before the build

Genuinely stop and make sure each of these is crisp. If one is fuzzy, that is the signal to re-read or ask.

1. Why does generating conversion code **at compile time**, rather than inspecting the value at runtime like Python does, let the compiler catch a whole class of bugs before the program runs?
2. What is a `derive` macro actually doing to your struct, mechanically? (If your answer is "it adds serialization," push further: *how*?)
3. Can you already see, at least in outline, why an **enum** plus `serde` would refuse an out-of-codebook value — even though we have not written that code yet?

{{#quiz ../../quizzes/01-data-model-concept-serde.toml}}

Next chapter is the first build: we create the workspace and the parameter types, and you write the first failing test.
