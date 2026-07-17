# Concept: Ownership and Moves

> **Kind:** Concept (read, do not code). **New to Rust: this is the chapter everything else assumes.**

Rust's one genuinely unfamiliar idea, the thing that has no direct equivalent in Python or most languages you have used, is **ownership**. Every other Rust concept in this course is ordinary once ownership is solid. So we go slow here, and we build a mental model you can actually reason with, not a rule you memorize.

## The one rule

Here is the whole thing in a sentence, from *Rust for Rustaceans*: <cite index="0-1">Rust's memory model centers on the idea that all values have a single owner — exactly one location, usually a scope, is responsible for ultimately deallocating each value.</cite>

Read that twice. Every value has **exactly one owner**. When the owner goes away (its scope ends), the value is cleaned up — "dropped" — automatically. No garbage collector deciding later, no manual `free`. The owner's scope ending *is* the cleanup signal.

## What "move" means

Now the consequence that trips up everyone new to Rust. What happens when you assign a value to a new variable, or pass it to a function? For most types, the value **moves**. The book again: <cite index="0-1">if the value is moved — by assigning it to a new variable, pushing it to a vector, or placing it on the heap — ownership moves from the old location to the new one, and you can no longer access the value through variables that flow from the original owner, even though the bits are technically still there.</cite>

This is the shock. In Python, `b = a` gives you two names for the same object. In Rust, for an owned type, `let b = a;` *moves* the value into `b`, and `a` is now unusable. Not copied — moved. The compiler will reject any later use of `a` with a clear error. Ownership transferred.

```rust
let s1 = String::from("hello");
let s2 = s1;          // the String moves from s1 into s2
// println!("{}", s1); // COMPILE ERROR: s1's value was moved into s2
println!("{}", s2);    // fine — s2 owns it now
```

<div class="callout predict">
<span class="callout-label">Why this rule exists</span>
Imagine both <code>s1</code> and <code>s2</code> stayed valid. When each went out of scope, each would try to free the same heap memory — a double-free, a classic memory-corruption bug. Move semantics make that impossible: after the move, only <code>s2</code> owns the memory, so only <code>s2</code> frees it. The rule is not bureaucracy; it is what lets Rust have no garbage collector and no double-frees at the same time.
</div>

## The escape hatch: Copy

Some small types do not move — they **copy**. The book calls them "rebels": <cite index="0-2">if a value's type implements the special Copy trait, the value is not considered to have moved even if reassigned; instead it is copied, and both locations remain accessible.</cite> Integers, floats, booleans, and small types made only of those are `Copy`. That is why this works fine:

```rust
let x = 42;
let y = x;             // i32 is Copy — x is COPIED into y
println!("{} {}", x, y); // both fine, both hold 42
```

The rule for what *can* be `Copy` is precise and worth internalizing: <cite index="0-2">to be Copy, it must be possible to duplicate the value simply by copying its bits — which excludes any type that owns a resource it must deallocate when dropped.</cite> A `String` owns heap memory, so it can never be `Copy`. An `i32` is just bits, so it is. This is exactly why, back in Part II, the small `Params` struct — made only of a `u8`, two simple enums, and a `bool` — can derive `Copy`, while a `Vignette` containing a `String` cannot.

## The flows mental model

The single most useful way to think about all of this, and the model the borrow checker actually uses internally, is **flows**. Picture each value as having a line — a flow — that starts when the value is created and traces through your program to its last use. A move ends one flow and starts another. Using a variable after its flow has ended (because the value moved away) is the error the compiler catches. Hold this picture; in the next chapter it makes borrowing click immediately.

## Questions to lock

Genuinely stop on each. This is the foundation.

1. In `let b = a;` where `a` is a `String`, what happens to `a`? Why? What would happen instead if `a` were an `i32`?
2. Why can a type that owns heap memory (like `String`) never be `Copy`?
3. What is the connection between "every value has one owner" and "Rust needs no garbage collector"?

{{#quiz ../../quizzes/00-orientation-ownership.toml}}

Next chapter: borrowing — how you let code *use* a value without taking ownership of it, which is what most function calls actually do.
