# Concept: Traits and the Config Boundary

> **Kind:** Concept.

## Traits, briefly

A **trait** is a set of behaviors a type promises to provide — Rust's version of an interface. `ScenarioFamily` will require any family type to answer `name()` and `is_valid(&Params)`. Code can then work with *any* family through the trait, without knowing which concrete family it is.

You have used traits already (`Serialize`, `Display` are traits). Here you *define* one for the first time in this project, which is a small but real step up.

Here is the complete mechanism on a toy domain (the same coffee shop from the iterators chapter): a trait, two types implementing it differently, and one generic function that works with both — and with any implementation written later:

```rust
#[derive(Debug, Clone, Copy)]
enum Size { Small, Large }
#[derive(Debug, Clone, Copy)]
enum Roast { Light, Dark }

struct Order { size: Size, roast: Roast, iced: bool }

/// The behavior contract: any house can say what it is called
/// and which orders it will actually make.
trait House {
    fn name(&self) -> &str;
    fn allows(&self, order: &Order) -> bool;
}

struct Downtown;
impl House for Downtown {
    fn name(&self) -> &str { "downtown" }
    fn allows(&self, o: &Order) -> bool {
        !(matches!(o.size, Size::Small) && o.iced) // house rule: no small iced drinks
    }
}

struct Airport;
impl House for Airport {
    fn name(&self) -> &str { "airport" }
    fn allows(&self, o: &Order) -> bool {
        !matches!(o.roast, Roast::Light) // house rule: dark roast only
    }
}

/// Generic over the trait: works with ANY house, present or future.
fn count_allowed(house: &impl House, orders: &[Order]) -> usize {
    orders.iter().filter(|o| house.allows(o)).count()
}

fn main() {
    let orders = [
        Order { size: Size::Small, roast: Roast::Light, iced: true },
        Order { size: Size::Large, roast: Roast::Light, iced: false },
        Order { size: Size::Large, roast: Roast::Dark, iced: true },
    ];
    println!("{}: {} of {} allowed", Downtown.name(), count_allowed(&Downtown, &orders), orders.len());
    println!("{}: {} of {} allowed", Airport.name(), count_allowed(&Airport, &orders), orders.len());
    // downtown: 2 of 3 allowed
    // airport: 1 of 3 allowed
}
```

The mapping is one-for-one: `House` ↔ `ScenarioFamily`, `allows` ↔ `is_valid`, `count_allowed(&impl House, …)` ↔ `generate(family: &impl ScenarioFamily, …)`. Each house's rule is ordinary Rust in its `impl` — typed, testable, compiled — which is exactly where the next section says validity logic belongs. Adding a second scenario family later is adding an `impl`, not touching `generate`.

## The design decision that matters

The consequential idea in this chapter is not the trait mechanics — it is **where the validity logic lives.** In the Python sketch, families carried a list of string "valid_rules" that were evaluated at runtime. That means: logic expressed as data, interpreted while the program runs, with errors surfacing only when a bad rule is hit.

We are making the opposite choice. **TOML holds content and metadata only** — the prompt template, the doctrine references, the action menu. The logic of *which parameter combinations are meaningful* lives in Rust code, in each family's `is_valid` implementation. That means the compiler checks it, and a malformed rule is a compile error, not a runtime surprise.

<div class="callout trap">
<span class="callout-label">The trap this avoids</span>
Putting logic in config feels flexible — "just edit the TOML, no recompile." But config-as-logic is untyped, unchecked, and evaluated late. For a reproducible instrument, you want the rules that shape your scenario set to be as scrutinized as any other code. Flexibility you cannot verify is a liability here, not a feature.
</div>

## Questions to lock

1. What does a trait let you do that you could not do by writing a function per concrete type?
2. Why is "which parameter combinations are valid" logic, and therefore code, rather than data that belongs in TOML?
3. What do you lose by moving that logic out of config, and why is the trade worth it for this project specifically?

{{#quiz ../../quizzes/02-generation-concept-traits.toml}}
