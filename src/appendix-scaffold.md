# Appendix: Workspace Scaffold

This is the orientation map for the *harness workspace you are building* (a separate repo from this book). It answers three questions the build chapters assume: **where does each file go, what is it called, and which dependencies does it need?** Every build chapter also carries its own Scaffold block; this page is the whole picture in one place.

> The harness workspace is its own project directory (e.g. `~/projects/rust/panoptes/`) — do not create it inside this course's repo.

## The full tree

Everything you create across the course, annotated with the chapter that creates it:

```
panoptes/
├── Cargo.toml                         # workspace manifest        (Part II · build-params)
├── crates/
│   ├── panoptes-core/                 # THE DATA MODEL — no I/O, no network, pure types
│   │   ├── Cargo.toml                 #                           (Part II · build-params)
│   │   └── src/
│   │       ├── lib.rs                 # re-exports                (Part II, grows each chapter)
│   │       ├── params.rs              # Params + parameter enums  (Part II · build-params)
│   │       ├── codes.rs               # codebook enums + CodedRow (Part II · build-codes)
│   │       ├── vignette.rs            # Vignette, vignette_id     (Part II · build-records)
│   │       └── record.rs              # ResponseRecord, Usage     (Part II · build-records)
│   ├── panoptes-gen/                  # STAGE 1 — generation
│   │   ├── Cargo.toml                 #                           (Part III · build-family)
│   │   └── src/
│   │       ├── lib.rs                 #                           (Part III · build-family)
│   │       ├── family.rs              # FamilySpec (TOML shape)   (Part III · build-family)
│   │       ├── validity.rs            # ScenarioFamily trait      (Part III · build-family)
│   │       ├── generate.rs            # grid → Vec<Vignette>      (Part III · build-generate)
│   │       └── main.rs                # `panoptes-gen` binary     (Part III · build-cli)
│   ├── panoptes-harness/              # STAGE 2 — execution
│   │   ├── Cargo.toml                 #                           (Part IV · build-client-log)
│   │   └── src/
│   │       ├── lib.rs                 #                           (Part IV · build-client-log)
│   │       ├── client.rs              # ModelClient trait         (Part IV · build-client-log)
│   │       ├── jsonl.rs               # append-only writer        (Part IV · build-client-log)
│   │       ├── anthropic.rs           # one provider impl         (Part IV · build-anthropic)
│   │       ├── dispatch.rs            # vignette×model×epoch loop (Part IV · build-dispatch)
│   │       └── main.rs                # `panoptes-run` binary     (Part IV · build-dispatch)
│   ├── panoptes-coding/               # STAGE 3 — coding I/O
│   │   ├── Cargo.toml                 #                           (Part V · build-sheets)
│   │   └── src/
│   │       ├── lib.rs                 #                           (Part V · build-sheets)
│   │       ├── sheets.rs              # blinded sheets + key      (Part V · build-sheets)
│   │       ├── validate.rs            # coded loader = validation (Part V · build-loader)
│   │       └── main.rs                # `panoptes-code` binary    (Part V · build-loader)
│   └── panoptes-cli/                  # the unified front door    (Part VII · build-cli)
│       ├── Cargo.toml
│       ├── src/main.rs                # `panoptes` subcommand enum
│       └── tests/pipeline.rs          # assert_cmd integration test
├── scenarios/
│   ├── families/ca_geo.toml           # content + metadata only   (Part III · build-family)
│   └── generated/                     # OUTPUT: manifest.csv + prompts/  (gitignore this)
├── harness/logs/                      # OUTPUT: responses.jsonl — append-only, sacred
├── coding/                            # OUTPUT: sheets/, coded/, blind_key.csv
├── reliability/README.md              # Stage-4 Python handoff    (Part VI · build-hardening)
└── README.md                          # run sequence + invariants (Part VI · build-hardening)
```

Build order matches the parts: core first (everything imports it), then gen → harness → coding in pipeline order, then the unified CLI on top.

## The workspace manifest — every dependency, declared once

The root `Cargo.toml` you write in the very first build chapter declares **all** shared dependencies under `[workspace.dependencies]`. Later crate manifests then just write `dep = { workspace = true }` — so if a chapter seems to use a crate "out of nowhere" (`strum`, `serde_with`, …), it was declared here on day one:

```toml
[workspace]
resolver = "2"
members = [
    "crates/panoptes-core",
    "crates/panoptes-gen",
    "crates/panoptes-harness",
    "crates/panoptes-coding",
]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }   # derive Serialize/Deserialize (everywhere)
serde_json = "1"        # JSON + JSONL encoding (records, tests)
toml = "1"              # family-spec parsing (Part III)
serde_with = "3"        # NoneAsEmptyString for optional free-text (Part II codes)
strum = { version = "0.26", features = ["derive"] } # enum ↔ string Display/EnumString (Part II)
tera = "1"              # prompt templating (Part III)
reqwest = { version = "0.12", features = ["json"] } # HTTP client (Part IV)
tokio = { version = "1", features = ["full"] }      # async runtime (Part IV)
async-trait = "0.1"     # async fn in the ModelClient trait (Part IV)
sha2 = "0.10"           # prompt hashing (Part III) + opaque response ids (Part IV)
chrono = { version = "0.4", features = ["serde"] }  # run_at timestamps (Part II records)
itertools = "0.13"      # iproduct! cartesian grid (Part III)
clap = { version = "4", features = ["derive"] }     # every CLI binary
anyhow = "1"            # application-level errors
thiserror = "2"         # typed library errors (Part V loader)
```

(Add `"crates/panoptes-cli"` to `members` when you reach Part VII.)

## Who depends on what

| Crate | `[dependencies]` | `[dev-dependencies]` |
|---|---|---|
| `panoptes-core` | serde, serde_with, strum, chrono | serde_json |
| `panoptes-gen` | panoptes-core, serde, toml, tera, sha2, itertools, clap, anyhow | serde_json, tempfile |
| `panoptes-harness` | panoptes-core, serde, serde_json, reqwest, tokio, async-trait, chrono, **sha2**, clap, anyhow | wiremock, tempfile |
| `panoptes-coding` | panoptes-core, serde, serde_json, **thiserror**, clap, anyhow | tempfile |
| `panoptes-cli` | panoptes-gen, panoptes-harness, panoptes-coding, clap, tokio, anyhow | assert_cmd, tempfile |

The two **bold** entries are corrections to the original task plan, which omitted them: `dispatch.rs` (Part IV) hashes response ids with `sha2`, and `validate.rs` (Part V) derives its error type with `thiserror`. Declare them when you create each crate's manifest and Parts IV–V will compile without surprises.

`wiremock = "0.6"`, `tempfile = "3"`, and `assert_cmd` are dev-dependencies declared per-crate (not in the workspace block).

## Expected test progression

A quick reality check for the end of each part — if your counts differ, look for a missed test, not a missed feature:

| After | `cargo test` shows |
|---|---|
| Part II complete | `-p panoptes-core`: 7 tests (2 params, 3 codes, 1 vignette, 1 record) |
| Part III complete | `-p panoptes-gen`: 10 tests (5 family + validity, 4 generate, 1 manifest) — and the binary prints `generated 18 vignettes → scenarios/generated` |
| Part IV complete | `-p panoptes-harness`: 4 tests (1 jsonl, 1 anthropic, 2 dispatch) |
| Part V complete | `-p panoptes-coding`: 5 tests (2 sheets, 3 validate) — and `panoptes-code --coded bad.jsonl` exits non-zero |
| Part VI complete | `cargo build --workspace && cargo test --workspace && cargo clippy --workspace` clean |
| Part VII complete | integration test green; `panoptes --help` lists generate / run / code |
