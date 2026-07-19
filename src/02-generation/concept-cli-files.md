# Concept: clap and Writing Files

> **Kind:** Concept. **New crate:** `clap` — plus the `std::fs` calls the build chapter needs. This chapter shows both working before you build with them.

## The idea

The next build turns your generation library into a *program*: a binary someone runs from a shell, pointing at a family spec, that leaves a manifest and prompt files on disk. That takes two skills the course has not taught yet — parsing command-line arguments and writing files — and this chapter covers exactly the slice of each you need.

## clap: your CLI is a struct

`clap`'s derive style inverts how you might expect argument parsing to work. You do not write parsing code and extract values from it — you **declare a struct that *is* your binary's interface**, and `#[derive(Parser)]` generates the parser from its shape at compile time. Fields become flags, field types become argument types, doc comments become help text:

```rust
use clap::Parser;
use std::path::PathBuf;

/// Generate vignettes from a family spec.
#[derive(Parser)]
#[command(name = "panoptes-gen")]
struct Cli {
    /// Path to the family TOML
    #[arg(long)]
    family: PathBuf,
    /// Output directory
    #[arg(long, default_value = "scenarios/generated")]
    out: PathBuf,
}

fn main() {
    // parse_from simulates: panoptes-gen --family scenarios/families/ca_geo.toml
    let cli = Cli::parse_from(["panoptes-gen", "--family", "scenarios/families/ca_geo.toml"]);
    println!("family = {}", cli.family.display());
    println!("out    = {} (defaulted)", cli.out.display());
}
```

```text
family = scenarios/families/ca_geo.toml
out    = scenarios/generated (defaulted)
```

Reading the attributes: `#[arg(long)]` makes a `--family <value>` flag; no default means it is **required**. `default_value` makes `--out` optional. The fields are `PathBuf` — a typed, owned filesystem path — so by the time your `main` body runs, the arguments already have the right types. (In the real binary you call `Cli::parse()`, which reads the actual command line; `parse_from` is the same machinery fed from code, handy in examples and tests.)

This is the same move the whole course keeps making: the interface is a **type**, and the compiler holds it. It is also serde's move — declare the shape, derive the machinery — pointed at argv instead of JSON.

## What the derive gives you for free

Run your compiled binary with `--help` and clap has already written the manual, straight from your doc comments:

```text
Generate vignettes from a family spec

Usage: panoptes-gen [OPTIONS] --family <FAMILY>

Options:
      --family <FAMILY>  Path to the family TOML
      --out <OUT>        Output directory [default: scenarios/generated]
  -h, --help             Print help
```

Forget a required flag and you get a real error, a usage reminder, and — note, ahead of Part VII — a **non-zero exit code** (clap uses `2`):

```text
error: the following required arguments were not provided:
  --family <FAMILY>

Usage: panoptes-gen --family <FAMILY>

For more information, try '--help'.
```

You wrote none of that. The struct declaration bought all of it.

## Writing files: three functions

Everything the build chapter does on disk is three `std::fs` calls — `create_dir_all` (make a directory, parents included, fine if it already exists), `fs::write` (create-or-truncate a file with the given bytes), and `read_to_string` (the whole file back as a `String`). Here they are producing exactly the build chapter's artifact — a manifest CSV built in memory, written, and read back:

```rust
use std::fs;

struct Row { id: String, confidence: u8 }

fn main() -> std::io::Result<()> {
    let rows = vec![
        Row { id: "ca_geo-030-H-REV-INFO".into(), confidence: 30 },
        Row { id: "ca_geo-060-D-IRREV-NOINFO".into(), confidence: 60 },
    ];

    // 1. Build the CSV in memory: header first, then one line per row.
    let mut csv = String::from("vignette_id,attribution_confidence\n");
    for r in &rows {
        csv.push_str(&format!("{},{}\n", r.id, r.confidence));
    }

    // 2. Write it. Create the directory first; fs::write creates or truncates the file.
    let dir = std::env::temp_dir().join("panoptes-demo");
    fs::create_dir_all(&dir)?;
    let path = dir.join("manifest.csv");
    fs::write(&path, &csv)?;

    // 3. Read it back to prove the round trip.
    let back = fs::read_to_string(&path)?;
    print!("{back}");
    println!("({} lines: 1 header + {} rows)", back.lines().count(), rows.len());
    Ok(())
}
```

```text
vignette_id,attribution_confidence
ca_geo-030-H-REV-INFO,30
ca_geo-060-D-IRREV-NOINFO,60
(3 lines: 1 header + 2 rows)
```

That is the entire `write_manifest` pattern: a `String` accumulated with `push_str`, one `format!` line per record, one `fs::write` at the end. Note `main` returning `std::io::Result<()>` so the `?` operator can propagate any I/O failure — and, per the previous chapters' theme, a failed write is then an *error*, not a silent absence. Prompt files are even simpler: one `fs::write(dir.join(format!("{id}.txt")), &prompt)` per vignette.

<div class="callout">
<span class="callout-label">CSV by hand — safe here, and when it stops being safe</span>
Real CSV has escaping rules (fields containing commas, quotes, newlines). We are writing it with <code>format!</code> anyway, because every manifest field is a <em>constrained token</em>: enum strings like <code>HOURS</code>, zero-padded numbers, a hex hash, an ID built from those. None can contain a comma. The prompt text — which absolutely could — deliberately lives in separate <code>.txt</code> files, not the manifest. If free text ever moves into the manifest, that is the moment to switch to the <code>csv</code> crate; the task plan's self-review flags exactly this.
</div>

## Mapping onto the build

The build chapter's `Cli` struct is the first example with the spec's names on it; `write_manifest` is the CSV example with the full seven-column header; the binary's last line prints the summary to **stderr** with `eprintln!` — stdout is kept clean, a habit that pays off when tools are chained in Part VII.

## Questions to lock

1. What does `#[derive(Parser)]` generate, and from what information? Where does the `--help` text come from?
2. What happens — message and exit code — when a required argument is missing, and who wrote that behavior?
3. Why is hand-built CSV acceptable for the manifest but not for the prompt text, and what is the signal to switch to the `csv` crate?
4. Why does the file-writing example's `main` return `std::io::Result<()>`, and what does `?` do with a failed write?

{{#quiz ../../quizzes/02-generation-concept-cli-files.toml}}
