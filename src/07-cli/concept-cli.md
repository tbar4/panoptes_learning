# Concept: Subcommands, Exit Codes, and Composability

> **Kind:** Concept.

You have three binaries — `panoptes-gen`, `panoptes-run`, `panoptes-code`. That works, but a real tool presents one front door: `panoptes generate`, `panoptes run`, `panoptes code`, like `git commit` and `git push`. This arc unifies them, and along the way teaches two things that make a command-line tool *well-behaved* rather than merely functional.

## Subcommands with clap

`clap` (which you already used for single binaries) models subcommands as an **enum**: each variant is a subcommand, each variant's fields are that subcommand's arguments. This is the same enum-as-closed-set idea from Part II, now applied to the shape of your CLI. The top-level parser dispatches on which variant it parsed, and you `match` on it to run the right stage. One binary, one help text, a discoverable set of verbs.

The whole mechanism, runnable (`parse_from` feeds argv in code, so you can watch the dispatch without a terminal):

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "panoptes")]
struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand)]
enum Command {
    /// Generate vignettes from a family spec
    Generate { #[arg(long)] family: String },
    /// Validate a coded file
    Code { #[arg(long)] coded: String },
}

fn main() {
    // parse_from simulates: panoptes code --coded primary.csv
    let cli = Cli::parse_from(["panoptes", "code", "--coded", "primary.csv"]);
    match cli.command {
        Command::Generate { family } => println!("would generate from {family}"),
        Command::Code { coded } => println!("would validate {coded}"),
    }
    // would validate primary.csv
}
```

The `match` is where the closed set pays off: add a `Report` variant later and every non-exhaustive `match` becomes a compile error until you handle it — the CLI cannot silently grow verbs the dispatch forgot.

## Exit codes: the part beginners skip

Here is the idea that separates a script from a tool. When your program finishes, it returns an **exit code** to the operating system — `0` for success, non-zero for failure. This is not decoration. From *Command-Line Rust*: <cite index="0-1">correctly reporting the exit status is a characteristic of well-behaved command-line programs. The exit value is important because a failed process used in conjunction with another process should cause the combination to fail.</cite>

Concretely, exit codes are what let programs **compose**. The book shows it with the shell's `&&`: <cite index="0-3">only if the first process reports success will the second process run.</cite> So this becomes possible:

```bash
panoptes generate --family ca_geo.toml && panoptes run --epochs 5
```

The `run` stage fires only if `generate` succeeded. If generation fails and exits non-zero, `run` never starts, and the whole line fails loudly. That is the behavior you want in a pipeline or a CI job — and it only works if each stage reports its status honestly.

<div class="callout">
<span class="callout-label">Why this matters for Panoptes specifically</span>
Recall <code>panoptes-code</code>, which validates coded data and exits non-zero on an off-codebook value. That exit code is what lets it become a CI gate: <code>panoptes code --coded primary.csv && deploy-analysis</code> refuses to run analysis on invalid data. The enums-as-validation property from Part II reaches its final form here — a validation failure becomes a process failure that stops the pipeline. The book's framing is exact: <cite index="0-4">ensuring that command-line programs correctly report errors makes them composable with other programs.</cite>
</div>

## Rust's `ExitCode` and `Result`-returning `main`

Modern Rust makes this ergonomic: `main` can return `Result<(), E>` (a non-`Ok` return becomes a non-zero exit automatically) or `std::process::ExitCode` for explicit control. You do not manage raw integers by hand; you return a type that carries success or failure, and the runtime translates it to the OS exit code. This is the same "push correctness into types" theme — even the process's success/failure is a typed value, not a convention you hope you remembered.

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    let coded_file_valid = false; // pretend validation just failed
    if coded_file_valid {
        ExitCode::SUCCESS
    } else {
        eprintln!("row 1: unknown variant `AGGRESSIVE`");
        ExitCode::FAILURE
    }
}
```

Run it in a shell and check: `echo $?` prints `1`, and `this-program && echo next` never prints `next`. The typed return value became the OS-level fact that composability is built on.

## Questions to lock

1. Why model subcommands as an enum, and how does that connect to the enums-as-validation idea from Part II?
2. What is an exit code, and why is "report a non-zero code on failure" the property that makes programs composable?
3. How does `panoptes code`'s non-zero exit on invalid data turn the codebook constraint into a pipeline gate?

{{#quiz ../../quizzes/07-cli-concept-cli.toml}}

Next: the build. We wrap the three stages behind one `panoptes` command and write an integration test that runs the whole pipeline end to end.
