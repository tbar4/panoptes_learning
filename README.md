# Building Panoptes — A Rust Harness, Test by Test

A phased, test-driven mdBook course for building the Panoptes evaluation harness in Rust.
Concept-first, then pair-programmed implementation, with **interactive quizzes** each phase.

## Quizzes — mdbook-quiz (three question types)
Quizzes use the [mdbook-quiz](https://github.com/cognitive-engineering-lab/mdbook-quiz) preprocessor:
- **MultipleChoice** — judgment questions (44)
- **ShortAnswer** — recall the exact term/keyword (7)
- **Tracing** — predict whether a snippet compiles and what it prints; **verified by the real Rust compiler at build time** (6)

Quiz sources live in `quizzes/*.toml`, one file per chapter, referenced from chapters via `{{#quiz ../../quizzes/NAME.toml}}`.

## Contents
- `src/` — all chapter markdown (39 chapters, 8 parts)
- `quizzes/` — 19 quiz TOML files, 57 questions total
- `theme/quiz-extra.css` — callout-box and inline-citation styling
- `book.toml` — mdBook + `[preprocessor.quiz]` config
- `.github/workflows/deploy.yml` — GitHub Pages CI (installs mdbook-quiz + Rust)
- `.gitignore` — ignores `book/` and the `src/mdbook-quiz/` scratch dir the preprocessor creates

## Build locally (requires Rust)
```bash
cargo install mdbook --locked
cargo install mdbook-quiz --locked      # needed for quizzes; Tracing questions invoke rustc
mdbook serve --open                      # live preview
mdbook build                             # outputs to book/
```

> This package ships **source only** — no pre-built `book/`. The Tracing questions must be
> compiled by mdbook-quiz on a machine with Rust, so the site is built at deploy time
> (see the included GitHub Actions workflow), not shipped pre-rendered.

## Deploy to GitHub Pages
Push to `main`. The included workflow installs Rust + mdbook + mdbook-quiz, runs `mdbook build`,
and publishes `book/` to Pages. In your repo settings, set Pages source to "GitHub Actions."

## Structure (8 parts, 39 chapters)
- **Part I — Foundations** (setup, ownership, borrowing) ← new-to-Rust starts here
- **Part II — Data Model** (serde, enums-as-validation)
- **Part III — Generation** (traits, iterators)
- **Part IV — Async** (futures, tokio, mock testing)
- **Part V — Coding** (blinding, parse-is-validation)
- **Part VI — Hardening** (lints, language boundary)
- **Part VII — CLI** (subcommands, exit codes, composability) ← unifies the binaries into one `panoptes` tool
- **Part VIII — Wrap-Up** (thesis integration, the Python tail)

## Authoring a quiz question
In `quizzes/NAME.toml`:
```toml
[[questions]]
type = "MultipleChoice"
prompt.prompt = "…"
prompt.distractors = ["wrong 1", "wrong 2"]
answer.answer = "the correct option"
context = "explanation shown after answering"

[[questions]]
type = "ShortAnswer"
prompt.prompt = "…"
answer.answer = "exact"
answer.alternatives = ["also accepted"]

[[questions]]
type = "Tracing"
prompt.program = """fn main() { /* ... */ }"""
answer.doesCompile = false          # or true, with answer.stdout = "..."
context = "why"
```
