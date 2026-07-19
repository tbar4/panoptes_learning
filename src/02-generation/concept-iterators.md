# Concept: Iterators and the Cartesian Product

> **Kind:** Concept. **New crate:** `itertools` — this chapter shows it working before you build with it.

## The idea

Generating vignettes means taking every combination of parameter values — 3 confidences × 2 time pressures × 2 reversibilities × 2 info-request states — and producing one vignette per valid combination. That "every combination" is a **cartesian product**, and Rust's iterator ecosystem expresses it cleanly: product → filter → map → collect, as a single chain.

That one sentence assumes a lot if iterators are new to you, so this chapter builds up to it with runnable code. Every example below runs — click the play button, or paste it into a scratch project.

## First, iterators themselves

An iterator is any value implementing the `Iterator` trait, which has one essential method: `next()`, returning `Some(item)` until the sequence is exhausted, then `None`. A `for` loop is sugar over exactly that. What makes iterators powerful is the **adapters** — methods like `map` and `filter` that wrap an iterator and return a new one — and the **consumers** like `collect` that drive the chain and produce a final value.

```rust
fn main() {
    let confidences = [30u8, 60, 95];

    let doubled: Vec<u8> = confidences.iter().map(|c| c * 2).collect();
    println!("{doubled:?}"); // [60, 120, 190]

    let high: Vec<u8> = confidences.iter().copied().filter(|&c| c >= 60).collect();
    println!("{high:?}"); // [60, 95]
}
```

Two things worth pausing on, because they will show up in your build:

- `.iter()` yields **references** (`&u8`), because iterating must not consume the array. `.copied()` converts `&u8` → `u8` for `Copy` types, which keeps the closures clean. Your parameter enums derive `Copy` for exactly this kind of ergonomic win.
- The closure `|&c| c >= 60` uses a **pattern** in its argument: `&c` matches a reference and binds the value. You will meet this again with tuples below.

## Laziness, demonstrated

Adapters do no work when you attach them. The chain is a *description* of a computation; a consumer like `collect()` is what actually pulls values through it. Watch the print order:

```rust
fn main() {
    let chain = [1, 2, 3].iter().map(|n| {
        println!("computing {n}");
        n * 10
    });
    println!("chain built — nothing has printed yet");

    let out: Vec<i32> = chain.collect();
    println!("{out:?}");
}
```

```text
chain built — nothing has printed yet
computing 1
computing 2
computing 3
[10, 20, 30]
```

The `map` closure runs *during* `collect`, not before. For generation this means the 24-combination grid never exists as a whole until you collect it — and if you only wanted the first valid vignette, `.find(...)` would stop the whole chain early, computing nothing else.

## What we are replacing: nested loops

Here is a cartesian product over a deliberately toy domain — coffee orders, so it cannot be mistaken for the answer key — written the obvious way:

```rust
#[derive(Debug, Clone, Copy)]
enum Size { Small, Large }
#[derive(Debug, Clone, Copy)]
enum Roast { Light, Dark }

fn main() {
    let mut combos = Vec::new();
    for size in [Size::Small, Size::Large] {
        for roast in [Roast::Light, Roast::Dark] {
            for iced in [true, false] {
                combos.push((size, roast, iced));
            }
        }
    }
    println!("{} combos, first = {:?}", combos.len(), combos[0]);
    // 8 combos, first = (Small, Light, true)
}
```

This works. Its weaknesses are structural: the *intent* ("every combination") is implicit in the indentation; the whole product is built eagerly into a `Vec` whether or not you need it all; and anything you do to each combination — filtering, transforming — has to happen inside the loop body, where it cannot be tested as a separate piece.

## `itertools` and `iproduct!`

`itertools` is a widely-used community crate that extends the standard iterator vocabulary with extra adapters and macros — it is to `Iterator` roughly what `serde_with` is to serde. It is already declared in `panoptes-gen`'s manifest. The piece you need is the `iproduct!` macro: give it N iterables and it yields **tuples of every combination, lazily**.

```rust
use itertools::iproduct;

#[derive(Debug, Clone, Copy)]
enum Size { Small, Large }
#[derive(Debug, Clone, Copy)]
enum Roast { Light, Dark }

fn main() {
    let combos: Vec<(Size, Roast, bool)> = iproduct!(
        [Size::Small, Size::Large],
        [Roast::Light, Roast::Dark],
        [true, false]
    ).collect();

    println!("{} combos, first = {:?}", combos.len(), combos[0]);
    // 8 combos, first = (Small, Light, true)
}
```

Same eight combinations, same order (rightmost dimension varies fastest, like an odometer) — but now the product is an *iterator*: a value you can pass around, chain adapters onto, and consume once at the end.

## The full shape: product → filter → map → collect

Now the pattern the build chapter asks of you, complete on the toy domain. One combination is nonsense for this shop — no small iced drinks — and each surviving combination becomes a typed struct:

```rust
use itertools::iproduct;

#[derive(Debug, Clone, Copy)]
enum Size { Small, Large }
#[derive(Debug, Clone, Copy)]
enum Roast { Light, Dark }

#[derive(Debug)]
struct Order { size: Size, roast: Roast, iced: bool }

fn valid(size: Size, iced: bool) -> bool {
    // house rule: no small iced drinks
    !(matches!(size, Size::Small) && iced)
}

fn main() {
    let orders: Vec<Order> = iproduct!(
        [Size::Small, Size::Large],
        [Roast::Light, Roast::Dark],
        [true, false]
    )
    .filter(|&(size, _roast, iced)| valid(size, iced))
    .map(|(size, roast, iced)| Order { size, roast, iced })
    .collect();

    println!("{} valid of 8 raw", orders.len()); // 6 valid of 8 raw
    println!("first = {:?}", orders[0]);
    // first = Order { size: Small, roast: Light, iced: false }
}
```

This maps one-for-one onto what you will write: the arrays become `CONFIDENCES` × the two parameter enums × the info-request bools (3 × 2 × 2 × 2 = 24); `valid` becomes the family's `is_valid` (CaGeo's rule cuts 24 to 18); `Order` becomes `Vignette` (with template rendering and hashing inside the `map`). The domain is different; the shape is identical.

<div class="callout trap">
<span class="callout-label">The tuple-destructuring trap</span>
<code>iproduct!</code> yields tuples, and the closures pattern-match them in their argument lists. Note the asymmetry above: <code>filter</code> lends each item to its closure by reference, so the pattern is <code>|&(size, _roast, iced)|</code> — the leading <code>&</code> matches the reference, and the copy is fine because everything is <code>Copy</code>. <code>map</code> consumes the item, so no <code>&</code>: <code>|(size, roast, iced)|</code>. Writing the wrong one produces a type error that looks scarier than it is; when you hit it, look at whether the adapter borrows or consumes.
</div>

## When the map can fail: collecting `Result`s

In the real `generate()`, the `map` step renders a template — which can fail — so the closure returns `anyhow::Result<Vignette>`, not `Vignette`. That would seem to leave you with an awkward `Vec<Result<…>>`, but `collect()` has a trick: it can target `Result<Vec<T>, E>` directly, succeeding only if every element succeeded and short-circuiting on the first error.

```rust
fn main() {
    let ok: Result<Vec<i32>, String> = ["1", "2", "3"].iter()
        .map(|s| s.parse::<i32>().map_err(|e| e.to_string()))
        .collect();
    println!("{ok:?}"); // Ok([1, 2, 3])

    let bad: Result<Vec<i32>, String> = ["1", "oops", "3"].iter()
        .map(|s| s.parse::<i32>().map_err(|e| e.to_string()))
        .collect();
    println!("{bad:?}"); // Err("invalid digit found in string")
}
```

This is why `generate()` can return `anyhow::Result<Vec<Vignette>>` from a single `collect()` call — one bad template render fails the whole generation loudly, instead of producing a partial scenario set silently.

## Why iterators, not loops

Now that you have seen both versions: the chain is **lazy** (nothing computes until consumed), **composable** (the validity predicate is a free function you can unit-test without running any product), and **legible** — "the cartesian product, minus invalid combos, each turned into a vignette" reads directly off the code, while the loop version buries that shape in indentation.

<div class="callout">
<span class="callout-label">Rust iterators vs. Python generators</span>
If you know Python generators, iterators will feel familiar: both are lazy sequences. The difference is that Rust iterators are statically typed and monomorphized — the compiler often turns a chain into code as tight as the hand-written loop, so you pay nothing for the abstraction.
</div>

## Questions to lock

1. What is a cartesian product, and why is it the right description of "all parameter combinations"?
2. What does "lazy" mean for an iterator chain, and why is `collect()` the moment the work actually happens? (Point to the line of output that proves it.)
3. In the coffee example, why does `filter`'s closure take `|&(size, _roast, iced)|` but `map`'s take `|(size, roast, iced)|`?
4. Why is a `filter` → `map` chain easier to test than the equivalent nested loops? (Where does `valid` live in each version?)
5. What type does `.collect::<Result<Vec<_>, _>>()` produce when one element fails, and why is that the right behavior for scenario generation?

{{#quiz ../../quizzes/02-generation-concept-iterators.toml}}
