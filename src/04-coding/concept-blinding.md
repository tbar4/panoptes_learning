# Concept: Blinding as a Correctness Property

> **Kind:** Concept.

## The problem, stated honestly

You are both the person who built the harness and the person who codes the responses. You have hypotheses about which model reasons which way. If your coding sheet showed you "this response came from Claude at 95% attribution confidence," your knowledge could unconsciously shape how you score — and a committee member could rightly ask whether your codes reflect the responses or your expectations.

**Blinding** removes the possibility. The coding sheet shows only an opaque `response_id` and the response text. No model name, no parameters. You code what is on the page, and only after coding is complete do you join back through the blind key to learn which response was which.

## Why this is a *type/code* property, not a discipline

You could try to blind yourself by "just not looking" at a column. That is a discipline, and disciplines fail. Instead we make it structural: the coding sheet is a type (`BlankSheetRow`) that *does not contain* the model or parameters. The blind key is a separate type (`KeyRow`) in a separate file that coders never open. The blinding is enforced by what the data structure *is*, not by what you remember to avoid.

Watch the property hold, mechanically — the sheet type simply has no field for what coders must not see, so serializing it *cannot* leak:

```rust
use serde::Serialize;

#[derive(Serialize)]
struct ResponseRecord {
    response_id: String,
    model: String,
    confidence: u8,
    response: String,
}

/// The sheet simply has no field for what coders must not see.
#[derive(Serialize)]
struct BlankSheetRow {
    response_id: String,
    response: String,
}

fn main() {
    let record = ResponseRecord {
        response_id: "3f9a1c07e2".into(),
        model: "anthropic/claude-x".into(),
        confidence: 95,
        response: "Recommend MONITOR.".into(),
    };
    let sheet = BlankSheetRow {
        response_id: record.response_id.clone(),
        response: record.response.clone(),
    };

    println!("record: {}", serde_json::to_string(&record).unwrap());
    let blind = serde_json::to_string(&sheet).unwrap();
    println!("sheet:  {blind}");
    println!("sheet mentions the model? {}", blind.contains("claude"));
}
```

```text
record: {"response_id":"3f9a1c07e2","model":"anthropic/claude-x","confidence":95,"response":"Recommend MONITOR."}
sheet:  {"response_id":"3f9a1c07e2","response":"Recommend MONITOR."}
sheet mentions the model? false
```

That final `false` is the whole chapter. There is no code path from `BlankSheetRow` to the model name, because the type has nowhere to put one — and the build chapter's `sheet_does_not_leak_identity` test asserts exactly this on the real types, so the property is not just structural but *guarded*.

<div class="callout">
<span class="callout-label">The through-line</span>
This is the same move as enums-as-validation, one level up. There, an invalid value could not be constructed. Here, a coding sheet that leaks identity cannot be constructed — the type has no field for it. Correctness properties belong in the types, where they hold whether or not anyone remembers them.
</div>

## The deterministic shuffle

There is a subtlety: even the *order* of responses could leak information (all of one model's responses appearing together). So sheets are shuffled — but with a seeded, deterministic shuffle, so the ordering is reproducible for the record while still breaking the model/parameter grouping. Reproducibility and blinding at once.

## Questions to lock

1. Why is blinding a genuine threat to the thesis specifically, given who the coder is?
2. What is the difference between blinding-by-discipline ("I won't look") and blinding-by-type ("the field does not exist"), and why does the second actually hold?
3. Why shuffle the sheet, and why must the shuffle be *deterministic* rather than truly random?

{{#quiz ../../quizzes/04-coding-concept-blinding.toml}}
