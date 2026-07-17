# Phase 0: The Toolchain and the Loop

Before any Panoptes code, we verify one thing: that you can write a failing test, see it fail legibly, fix it, and see it pass. The entire course runs on that loop being fast and trustworthy. This phase is deliberately trivial — that is the point. We are testing the machinery, not your ability.

## Objective

- Confirm a working Rust toolchain (`rustc`, `cargo`).
- Create a throwaway crate.
- Write one deliberately failing test, watch it fail, fix it, watch it pass.
- Internalize what `cargo test` output looks like when things break.

## Install the toolchain

Rust is installed through `rustup`, which manages compiler versions:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# accept the defaults, then:
source "$HOME/.cargo/env"
rustc --version
cargo --version
```

You want a recent stable toolchain. If you already have one, `rustup update` brings it current.

## Create a throwaway crate

```bash
cargo new hello-loop
cd hello-loop
```

`cargo new` scaffolds a tiny package: a `Cargo.toml` manifest and a `src/main.rs`. Open `src/main.rs`. Replace its contents with a single function and a test:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("{}", add(2, 2));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn two_plus_two_is_five() {
        assert_eq!(add(2, 2), 5); // deliberately wrong
    }
}
```

## The failing run

<div class="callout predict">
<span class="callout-label">Predict first</span>
Before running it: what will <code>cargo test</code> print? Not just "it fails" — what specifically will it show you about <em>why</em>? Where will the line number point? Decide, then run.
</div>

```bash
cargo test
```

You will see something close to this:

```
running 1 test
test tests::two_plus_two_is_five ... FAILED

failures:

---- tests::two_plus_two_is_five stdout ----
thread 'tests::two_plus_two_is_five' panicked at src/main.rs:15:9:
assertion `left == right` failed
  left: 4
 right: 5
```

Read that carefully, because you will read a hundred of these. It tells you the exact test, the exact file and line, and — critically — `left: 4, right: 5`, the actual value against the expected value. `assert_eq!` always reports both sides. This is why we write assertions with the computed value on the left and the expected on the right: the output reads naturally.

## The fix

Change the `5` to a `4`:

```rust
        assert_eq!(add(2, 2), 4);
```

Run again:

```
test tests::two_plus_two_is_five ... ok

test result: ok. 1 passed; 0 failed; 0 ignored
```

That is the loop. Every build chapter is this, with more interesting types in the middle.

## What just happened, named

- `#[cfg(test)]` means "only compile this module when running tests" — your test code is not in your shipped binary.
- `mod tests { }` is a module, a namespace. Tests conventionally live in a nested `tests` module.
- `use super::*;` pulls in everything from the parent module so the test can see `add`.
- `#[test]` marks a function as a test case for `cargo test` to discover and run.

Every one of these appears in the very first Panoptes task, so you have now seen the whole skeleton.

{{#quiz ../../quizzes/00-orientation-phase-0-setup.toml}}

## Done when

You can run `cargo test`, read a failure, and fix it without thinking about the mechanics. When that loop is automatic, turn the page: Part II begins with the concept that makes the whole harness worth building in Rust.
