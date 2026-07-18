# Appendix: The Answer Key (Full Task Plan)

This is the complete implementation plan the build chapters are drawn from — every manifest, type definition, attribute, test, and expected output, in full. Each build chapter names its task (**Maps to: Task N**); come here when you want the exact spec or to compare your implementation after your own attempt.

**Use it as an answer key, not a script.** The learning is in writing the tests and implementation yourself first. But design decisions — field lists, string conventions, wire formats — are *givens*, not puzzles: look them up here freely.

**Goal:** A typed, reproducible Rust harness that generates parameterized scenario vignettes, dispatches them to version-pinned model APIs, logs every raw response to an append-only store, and holds human-coded results in types that reject invalid codebook values at parse time. Stages 5–6 (analysis, reliability stats) stay in Python and read this harness's output files.

**Architecture:** A Cargo workspace with four member crates split by responsibility: `panoptes-core` (shared data model — the types every other crate depends on), `panoptes-gen` (Stage 1 scenario generation), `panoptes-harness` (Stage 2 execution), and `panoptes-coding` (Stage 3 coding-sheet I/O with type-enforced validation). Stage 4 reliability and Stages 5–6 analysis are Python, consuming the JSONL/CSV files this workspace produces. The interface between Rust and Python is the file contract (`responses.jsonl`, `coded/*.csv`), never a language binding.

**Tech Stack:** Rust 2021, `serde` + `serde_json` + `toml` (data model + file I/O; TOML for hand-authored family specs — the maintained `toml` crate replaces the archived `serde_yaml`), `serde_with` (empty-string rejection), `strum` (enum↔string), `tera` (prompt templating), `reqwest` + `tokio` + `async-trait` (async API dispatch), `sha2` (prompt hashing), `chrono` (timestamps), `itertools` (cartesian product), `clap` (CLI), `anyhow` + `thiserror` (errors). Testing: built-in `cargo test` + `wiremock` (HTTP mocking), `tempfile` (filesystem tests), `assert_cmd` (CLI integration).

---

## File Structure

```
panoptes/
├── Cargo.toml                         # workspace manifest
├── crates/
│   ├── panoptes-core/                 # THE DATA MODEL — no I/O, no network, pure types
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs                 # re-exports
│   │       ├── params.rs              # Params + parameter enums (TimePressure, Reversibility)
│   │       ├── codes.rs               # codebook enums (StrategicLogic, ActionType, ...) + CodedRow
│   │       ├── vignette.rs            # Vignette, VignetteId
│   │       └── record.rs              # ResponseRecord, Usage (the JSONL row)
│   ├── panoptes-gen/                  # STAGE 1
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── family.rs              # FamilySpec (TOML shape) + ScenarioFamily trait
│   │       ├── validity.rs            # per-family validity rules (typed, not config strings)
│   │       ├── generate.rs            # cartesian product → Vec<Vignette> + manifest
│   │       └── main.rs                # `panoptes-gen` CLI binary
│   ├── panoptes-harness/              # STAGE 2
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── client.rs              # ModelClient trait
│   │       ├── anthropic.rs           # one provider impl (pattern for others)
│   │       ├── dispatch.rs            # the vignette × model × epoch loop
│   │       ├── jsonl.rs               # append-only writer
│   │       └── main.rs                # `panoptes-run` CLI binary
│   └── panoptes-coding/               # STAGE 3
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── sheets.rs              # blinded blank-sheet generation + blind key
│           ├── validate.rs            # load coded CSV → CodedRow (parse = validation)
│           └── main.rs                # `panoptes-code` CLI binary
├── scenarios/
│   ├── families/ca_geo.toml           # content + metadata only
│   └── generated/                     # OUTPUT: manifest.csv + prompts/
├── harness/logs/                      # OUTPUT: responses.jsonl (append-only, sacred)
├── coding/                            # OUTPUT: sheets/, coded/, blind_key.csv
└── analysis/                          # Python (Stage 5–6), reads the above — not built here
```

**Decomposition rationale.** `panoptes-core` holds every type crossing a crate boundary, so the data model is defined exactly once and the compiler enforces consistency everywhere downstream — a `StrategicLogic` variant added in core is instantly visible to coding and analysis-export. The three stage crates depend on core and on each other only through core's types, never directly. Files split by responsibility: parameter types, code types, and record types are separate files in core because they change for different reasons (a new scenario parameter vs. a new codebook criterion vs. a new logged field).

**Build order.** Core first (everything depends on it), then the three stages in pipeline order (gen → harness → coding). Stage 4 reliability is Python and is scaffolded in the final task as a file-contract stub, not implemented here.

---

## Task 1: Workspace + core parameter types

**Files:**
- Create: `Cargo.toml` (workspace root)
- Create: `crates/panoptes-core/Cargo.toml`
- Create: `crates/panoptes-core/src/lib.rs`
- Create: `crates/panoptes-core/src/params.rs`

- [ ] **Step 1: Create the workspace manifest**

`Cargo.toml`:
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
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "1"
serde_with = "3"
strum = { version = "0.26", features = ["derive"] }
tera = "1"
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"
sha2 = "0.10"
chrono = { version = "0.4", features = ["serde"] }
itertools = "0.13"
clap = { version = "4", features = ["derive"] }
anyhow = "1"
thiserror = "2"
```

- [ ] **Step 2: Create the core crate manifest**

`crates/panoptes-core/Cargo.toml`:
```toml
[package]
name = "panoptes-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { workspace = true }
serde_with = { workspace = true }
strum = { workspace = true }
chrono = { workspace = true }
```

- [ ] **Step 3: Write the failing test for parameter enum round-tripping**

`crates/panoptes-core/src/params.rs`:
```rust
use serde::{Deserialize, Serialize};
use strum::{Display, EnumString};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Display, EnumString)]
#[strum(serialize_all = "UPPERCASE")]
pub enum TimePressure { Hours, Days }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Display, EnumString)]
#[strum(serialize_all = "UPPERCASE")]
pub enum Reversibility { Reversible, Irreversible }

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub struct Params {
    pub attribution_confidence: u8,
    pub time_pressure: TimePressure,
    pub reversibility: Reversibility,
    pub info_request: bool,
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::str::FromStr;

    #[test]
    fn time_pressure_string_roundtrip() {
        assert_eq!(TimePressure::Hours.to_string(), "HOURS");
        assert_eq!(TimePressure::from_str("DAYS").unwrap(), TimePressure::Days);
    }

    #[test]
    fn params_json_roundtrip() {
        let p = Params { attribution_confidence: 60, time_pressure: TimePressure::Hours,
            reversibility: Reversibility::Irreversible, info_request: true };
        let json = serde_json::to_string(&p).unwrap();
        let back: Params = serde_json::from_str(&json).unwrap();
        assert_eq!(p, back);
    }
}
```

- [ ] **Step 4: Write lib.rs re-exporting the module**

`crates/panoptes-core/src/lib.rs`:
```rust
pub mod params;
pub use params::{Params, Reversibility, TimePressure};
```

Add `serde_json` as a dev-dependency in `crates/panoptes-core/Cargo.toml`:
```toml
[dev-dependencies]
serde_json = { workspace = true }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cargo test -p panoptes-core params`
Expected: PASS, 2 tests (`time_pressure_string_roundtrip`, `params_json_roundtrip`)

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml crates/panoptes-core
git commit -m "feat(core): workspace + typed scenario parameters"
```

---

## Task 2: Core codebook types (the validation-by-type payoff)

**Files:**
- Create: `crates/panoptes-core/src/codes.rs`
- Modify: `crates/panoptes-core/src/lib.rs`

- [ ] **Step 1: Write the failing test — invalid codebook value must fail to deserialize**

`crates/panoptes-core/src/codes.rs`:
```rust
use serde::{Deserialize, Serialize};
use serde_with::{serde_as, NoneAsEmptyString};
use strum::{Display, EnumString};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Display, EnumString)]
#[strum(serialize_all = "SCREAMING_SNAKE_CASE")]
pub enum ActionType {
    SensorRetask, Maneuver, Monitor, EscalateToCommand, RequestData, NoAction,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Display, EnumString)]
#[strum(serialize_all = "SCREAMING_SNAKE_CASE")]
pub enum StrategicLogic {
    Control, Maritime, Political, Procedural, None, Mixed,
}

/// One human-coded row. Bad enum values are a *deserialization error*,
/// not something a separate lint step has to catch.
#[serde_as]
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct CodedRow {
    pub response_id: String,
    pub c1_action: ActionType,
    pub c2_logic: StrategicLogic,
    pub c2_anchor_quote: String,
    pub c3_escalation: u8,
    pub codebook_version: String,
    #[serde_as(as = "NoneAsEmptyString")]
    pub coder_notes: Option<String>,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_logic_parses() {
        assert_eq!(serde_json::from_str::<StrategicLogic>("\"CONTROL\"").unwrap(),
                   StrategicLogic::Control);
    }

    #[test]
    fn invalid_logic_is_rejected() {
        // "AGGRESSIVE" is not in the codebook — must NOT silently accept
        let r = serde_json::from_str::<StrategicLogic>("\"AGGRESSIVE\"");
        assert!(r.is_err(), "invalid codebook value must fail to parse");
    }

    #[test]
    fn coded_row_json_roundtrip() {
        let row = CodedRow {
            response_id: "abc123".into(), c1_action: ActionType::SensorRetask,
            c2_logic: StrategicLogic::Control, c2_anchor_quote: "positional advantage".into(),
            c3_escalation: 0, codebook_version: "0.3".into(), coder_notes: None,
        };
        let json = serde_json::to_string(&row).unwrap();
        assert_eq!(serde_json::from_str::<CodedRow>(&json).unwrap(), row);
    }
}
```

- [ ] **Step 2: Re-export from lib.rs**

Append to `crates/panoptes-core/src/lib.rs`:
```rust
pub mod codes;
pub use codes::{ActionType, CodedRow, StrategicLogic};
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `cargo test -p panoptes-core codes`
Expected: PASS, 3 tests. Critically `invalid_logic_is_rejected` proves the codebook constraint lives in the type.

- [ ] **Step 4: Commit**

```bash
git add crates/panoptes-core
git commit -m "feat(core): codebook enums with parse-time validation"
```

---

## Task 3: Core record + vignette types (the file contract)

**Files:**
- Create: `crates/panoptes-core/src/vignette.rs`
- Create: `crates/panoptes-core/src/record.rs`
- Modify: `crates/panoptes-core/src/lib.rs`

- [ ] **Step 1: Write the failing test for VignetteId formatting**

`crates/panoptes-core/src/vignette.rs`:
```rust
use crate::params::Params;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Vignette {
    pub id: String,
    pub family: String,
    pub params: Params,
    pub prompt: String,
    pub prompt_sha256: String,
}

/// Deterministic ID: family-<conf>-<H|D>-<REV|IRREV>-<INFO|NOINFO>.
pub fn vignette_id(family: &str, p: &Params) -> String {
    let tp = if matches!(p.time_pressure, crate::params::TimePressure::Hours) { "H" } else { "D" };
    let rv = if matches!(p.reversibility, crate::params::Reversibility::Reversible) { "REV" } else { "IRREV" };
    let info = if p.info_request { "INFO" } else { "NOINFO" };
    format!("{}-{:03}-{}-{}-{}", family, p.attribution_confidence, tp, rv, info)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::params::{Reversibility, TimePressure};

    #[test]
    fn id_is_deterministic_and_formatted() {
        let p = Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
            reversibility: Reversibility::Reversible, info_request: true };
        assert_eq!(vignette_id("ca_geo", &p), "ca_geo-030-H-REV-INFO");
    }
}
```

- [ ] **Step 2: Write the failing test for ResponseRecord round-trip**

`crates/panoptes-core/src/record.rs`:
```rust
use crate::params::Params;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}

/// One line in responses.jsonl — the dataset of record. Append-only.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct ResponseRecord {
    pub response_id: String,
    pub vignette_id: String,
    pub model: String,
    pub epoch: u32,
    pub params: Params,
    pub prompt: String,
    pub response: String,
    pub usage: Usage,
    pub run_at: DateTime<Utc>,
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::params::{Reversibility, TimePressure};

    #[test]
    fn record_jsonl_roundtrip() {
        let rec = ResponseRecord {
            response_id: "r1".into(), vignette_id: "ca_geo-030-H-REV-INFO".into(),
            model: "anthropic/claude-x".into(), epoch: 0,
            params: Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
                reversibility: Reversibility::Reversible, info_request: true },
            prompt: "p".into(), response: "resp".into(),
            usage: Usage { input_tokens: 10, output_tokens: 20 },
            run_at: "2026-07-17T12:00:00Z".parse().unwrap(),
        };
        let line = serde_json::to_string(&rec).unwrap();
        assert_eq!(serde_json::from_str::<ResponseRecord>(&line).unwrap(), rec);
    }
}
```

- [ ] **Step 3: Re-export from lib.rs**

Append to `crates/panoptes-core/src/lib.rs`:
```rust
pub mod vignette;
pub mod record;
pub use vignette::{vignette_id, Vignette};
pub use record::{ResponseRecord, Usage};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cargo test -p panoptes-core`
Expected: PASS, all core tests (params + codes + vignette + record).

- [ ] **Step 5: Commit**

```bash
git add crates/panoptes-core
git commit -m "feat(core): vignette + response record types (the file contract)"
```

---

## Task 4: Family spec loading + validity trait (Stage 1a)

**Files:**
- Create: `crates/panoptes-gen/Cargo.toml`
- Create: `crates/panoptes-gen/src/lib.rs`
- Create: `crates/panoptes-gen/src/family.rs`
- Create: `crates/panoptes-gen/src/validity.rs`
- Create: `scenarios/families/ca_geo.toml`

- [ ] **Step 1: Create the gen crate manifest**

`crates/panoptes-gen/Cargo.toml`:
```toml
[package]
name = "panoptes-gen"
version = "0.1.0"
edition = "2021"

[dependencies]
panoptes-core = { path = "../panoptes-core" }
serde = { workspace = true }
toml = { workspace = true }
tera = { workspace = true }
sha2 = { workspace = true }
itertools = { workspace = true }
clap = { workspace = true }
anyhow = { workspace = true }

[dev-dependencies]
serde_json = { workspace = true }
```

- [ ] **Step 2: Create the example family TOML (content + metadata only)**

`scenarios/families/ca_geo.toml`:
```toml
name = "ca_geo"
version = 1
title = "Ambiguous close-approach, GEO"
doctrine_refs = ["JP 3-14", "Space Capstone Publication"]
action_menu = [
  "SENSOR_RETASK",
  "MANEUVER",
  "MONITOR",
  "ESCALATE_TO_COMMAND",
]
template = """
You are the space operations duty officer. A close-approach event has been
flagged in the GEO belt. Attribution confidence: {{ attribution_confidence }}%.
Time to closest approach: {{ time_pressure }}.
{% if info_request %}You may request additional sensor tasking before deciding.{% endif %}
Available actions: {{ action_menu }}.
State your recommended action and justification.
"""
```

- [ ] **Step 3: Write the failing test for FamilySpec TOML parsing**

`crates/panoptes-gen/src/family.rs`:
```rust
use serde::Deserialize;

#[derive(Debug, Clone, Deserialize)]
pub struct FamilySpec {
    pub name: String,
    pub version: u32,
    pub title: String,
    pub doctrine_refs: Vec<String>,
    pub action_menu: Vec<String>,
    pub template: String,
}

impl FamilySpec {
    pub fn from_toml(s: &str) -> anyhow::Result<Self> {
        Ok(toml::from_str(s)?)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    const SPEC: &str = r#"
name = "ca_geo"
version = 1
title = "Ambiguous close-approach, GEO"
doctrine_refs = ["JP 3-14"]
action_menu = ["SENSOR_RETASK", "MONITOR"]
template = "confidence {{ attribution_confidence }}"
"#;

    #[test]
    fn parses_family_spec() {
        let spec = FamilySpec::from_toml(SPEC).unwrap();
        assert_eq!(spec.name, "ca_geo");
        assert_eq!(spec.version, 1);
        assert_eq!(spec.action_menu.len(), 2);
        assert!(spec.doctrine_refs.contains(&"JP 3-14".to_string()));
    }
}
```

- [ ] **Step 4: Write the failing test for the validity trait**

`crates/panoptes-gen/src/validity.rs`:
```rust
use panoptes_core::params::{Params, TimePressure};

/// Per-family logic for which parameter combinations are meaningful.
/// Lives in Rust, not in config strings — no runtime rule evaluation.
pub trait ScenarioFamily {
    fn name(&self) -> &str;
    fn is_valid(&self, p: &Params) -> bool;
}

pub struct CaGeo;

impl ScenarioFamily for CaGeo {
    fn name(&self) -> &str { "ca_geo" }
    fn is_valid(&self, p: &Params) -> bool {
        // Exclude an HOURS window with no info-request option as nonsensical
        // for this family (no time to act AND no way to gather data).
        !(matches!(p.time_pressure, TimePressure::Hours) && !p.info_request)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use panoptes_core::params::{Reversibility, TimePressure};

    fn p(tp: TimePressure, info: bool) -> Params {
        Params { attribution_confidence: 60, time_pressure: tp,
            reversibility: Reversibility::Reversible, info_request: info }
    }

    #[test]
    fn rejects_excluded_combo() {
        assert!(!CaGeo.is_valid(&p(TimePressure::Hours, false)));
    }

    #[test]
    fn accepts_valid_combo() {
        assert!(CaGeo.is_valid(&p(TimePressure::Hours, true)));
        assert!(CaGeo.is_valid(&p(TimePressure::Days, false)));
    }
}
```

- [ ] **Step 5: Write lib.rs**

`crates/panoptes-gen/src/lib.rs`:
```rust
pub mod family;
pub mod validity;
pub use family::FamilySpec;
pub use validity::{CaGeo, ScenarioFamily};
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cargo test -p panoptes-gen family && cargo test -p panoptes-gen validity`
Expected: PASS, 5 tests total.

- [ ] **Step 7: Commit**

```bash
git add crates/panoptes-gen scenarios/families/ca_geo.toml
git commit -m "feat(gen): family spec loading + typed validity rules"
```

---

## Task 5: Vignette generation + manifest (Stage 1b)

**Files:**
- Create: `crates/panoptes-gen/src/generate.rs`
- Modify: `crates/panoptes-gen/src/lib.rs`

- [ ] **Step 1: Write the failing test for the parameter grid product**

`crates/panoptes-gen/src/generate.rs`:
```rust
use itertools::iproduct;
use panoptes_core::params::{Params, Reversibility, TimePressure};
use panoptes_core::vignette::{vignette_id, Vignette};
use panoptes_core::Vignette as _;
use crate::family::FamilySpec;
use crate::validity::ScenarioFamily;
use sha2::{Digest, Sha256};
use tera::{Context, Tera};

const CONFIDENCES: [u8; 3] = [30, 60, 95];

pub fn all_params() -> impl Iterator<Item = Params> {
    iproduct!(
        CONFIDENCES,
        [TimePressure::Hours, TimePressure::Days],
        [Reversibility::Reversible, Reversibility::Irreversible],
        [true, false]
    ).map(|(ac, tp, rv, info)| Params {
        attribution_confidence: ac, time_pressure: tp, reversibility: rv, info_request: info,
    })
}

pub fn generate(family: &impl ScenarioFamily, spec: &FamilySpec) -> anyhow::Result<Vec<Vignette>> {
    let mut tera = Tera::default();
    tera.add_raw_template(&spec.name, &spec.template)?;

    all_params()
        .filter(|p| family.is_valid(p))
        .map(|p| {
            let mut ctx = Context::new();
            ctx.insert("attribution_confidence", &p.attribution_confidence);
            ctx.insert("time_pressure", &p.time_pressure.to_string());
            ctx.insert("info_request", &p.info_request);
            ctx.insert("action_menu", &spec.action_menu.join(", "));
            let prompt = tera.render(&spec.name, &ctx)?;
            let hash = format!("{:x}", Sha256::digest(prompt.as_bytes()));
            Ok(Vignette {
                id: vignette_id(&spec.name, &p), family: spec.name.clone(),
                params: p, prompt, prompt_sha256: hash,
            })
        })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::validity::CaGeo;

    fn spec() -> FamilySpec {
        FamilySpec::from_toml(r#"
name = "ca_geo"
version = 1
title = "t"
doctrine_refs = []
action_menu = ["MONITOR"]
template = "conf {{ attribution_confidence }} info {{ info_request }}"
"#).unwrap()
    }

    #[test]
    fn grid_has_24_raw_combinations() {
        // 3 conf × 2 time × 2 rev × 2 info = 24 before validity filter
        assert_eq!(all_params().count(), 24);
    }

    #[test]
    fn validity_filter_reduces_count() {
        let vs = generate(&CaGeo, &spec()).unwrap();
        // CaGeo excludes HOURS+NOINFO: removes 3 conf × 2 rev = 6 → 18 remain
        assert_eq!(vs.len(), 18);
    }

    #[test]
    fn ids_are_unique() {
        let vs = generate(&CaGeo, &spec()).unwrap();
        let mut ids: Vec<_> = vs.iter().map(|v| v.id.clone()).collect();
        ids.sort();
        ids.dedup();
        assert_eq!(ids.len(), vs.len(), "vignette IDs must be unique");
    }

    #[test]
    fn template_renders_params() {
        let vs = generate(&CaGeo, &spec()).unwrap();
        let v = vs.iter().find(|v| v.params.attribution_confidence == 30).unwrap();
        assert!(v.prompt.contains("conf 30"));
    }
}
```

Note: remove the erroneous `use panoptes_core::Vignette as _;` line — it's shown here only to flag that the working import is `use panoptes_core::vignette::{vignette_id, Vignette};`. Delete the duplicate.

- [ ] **Step 2: Fix imports and re-export from lib.rs**

Correct the imports at the top of `generate.rs` to exactly:
```rust
use itertools::iproduct;
use panoptes_core::params::{Params, Reversibility, TimePressure};
use panoptes_core::vignette::{vignette_id, Vignette};
use crate::family::FamilySpec;
use crate::validity::ScenarioFamily;
use sha2::{Digest, Sha256};
use tera::{Context, Tera};
```

Append to `crates/panoptes-gen/src/lib.rs`:
```rust
pub mod generate;
pub use generate::{all_params, generate};
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `cargo test -p panoptes-gen generate`
Expected: PASS, 4 tests. `validity_filter_reduces_count` and `ids_are_unique` are the load-bearing ones.

- [ ] **Step 4: Commit**

```bash
git add crates/panoptes-gen
git commit -m "feat(gen): parameter-grid vignette generation with unique IDs"
```

---

## Task 6: Generation CLI + manifest writer (Stage 1c)

**Files:**
- Create: `crates/panoptes-gen/src/main.rs`
- Modify: `crates/panoptes-gen/Cargo.toml` (add `[[bin]]`)

- [ ] **Step 1: Declare the binary in the manifest**

Append to `crates/panoptes-gen/Cargo.toml`:
```toml
[[bin]]
name = "panoptes-gen"
path = "src/main.rs"
```

- [ ] **Step 2: Write the manifest writer with a test**

`crates/panoptes-gen/src/main.rs`:
```rust
use anyhow::Result;
use clap::Parser;
use panoptes_core::vignette::Vignette;
use panoptes_gen::{generate, CaGeo, FamilySpec};
use std::fs;
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// Path to the family TOML
    #[arg(long)]
    family: PathBuf,
    /// Output directory (manifest.csv + prompts/ written here)
    #[arg(long, default_value = "scenarios/generated")]
    out: PathBuf,
}

/// Serialize vignettes to a manifest CSV (one row per vignette, no prompt body).
fn write_manifest(vignettes: &[Vignette], path: &std::path::Path) -> Result<()> {
    let mut w = String::from("vignette_id,family,attribution_confidence,time_pressure,reversibility,info_request,prompt_sha256\n");
    for v in vignettes {
        w.push_str(&format!("{},{},{},{},{},{},{}\n",
            v.id, v.family, v.params.attribution_confidence,
            v.params.time_pressure, v.params.reversibility, v.params.info_request,
            v.prompt_sha256));
    }
    fs::write(path, w)?;
    Ok(())
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let spec = FamilySpec::from_toml(&fs::read_to_string(&cli.family)?)?;
    let vignettes = generate(&CaGeo, &spec)?;

    fs::create_dir_all(cli.out.join("prompts"))?;
    for v in &vignettes {
        fs::write(cli.out.join("prompts").join(format!("{}.txt", v.id)), &v.prompt)?;
    }
    write_manifest(&vignettes, &cli.out.join("manifest.csv"))?;
    eprintln!("generated {} vignettes → {}", vignettes.len(), cli.out.display());
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use panoptes_core::params::{Params, Reversibility, TimePressure};

    #[test]
    fn manifest_has_header_and_row_per_vignette() {
        let v = Vignette {
            id: "ca_geo-030-H-REV-INFO".into(), family: "ca_geo".into(),
            params: Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
                reversibility: Reversibility::Reversible, info_request: true },
            prompt: "p".into(), prompt_sha256: "deadbeef".into(),
        };
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("manifest.csv");
        write_manifest(&[v], &path).unwrap();
        let contents = std::fs::read_to_string(&path).unwrap();
        assert!(contents.starts_with("vignette_id,family,"));
        assert_eq!(contents.lines().count(), 2); // header + 1 row
        assert!(contents.contains("ca_geo-030-H-REV-INFO"));
    }
}
```

- [ ] **Step 3: Add `tempfile` dev-dependency**

Append to `crates/panoptes-gen/Cargo.toml` `[dev-dependencies]`:
```toml
tempfile = "3"
```

- [ ] **Step 4: Run the test, then run the binary end-to-end**

Run: `cargo test -p panoptes-gen manifest`
Expected: PASS, 1 test.

Run: `cargo run -p panoptes-gen -- --family scenarios/families/ca_geo.toml`
Expected: stderr `generated 18 vignettes → scenarios/generated`, and `scenarios/generated/manifest.csv` + 18 files in `scenarios/generated/prompts/` exist.

- [ ] **Step 5: Commit**

```bash
git add crates/panoptes-gen
git commit -m "feat(gen): CLI binary writing manifest + prompt files"
```

---

## Task 7: ModelClient trait + append-only JSONL writer (Stage 2a)

**Files:**
- Create: `crates/panoptes-harness/Cargo.toml`
- Create: `crates/panoptes-harness/src/lib.rs`
- Create: `crates/panoptes-harness/src/client.rs`
- Create: `crates/panoptes-harness/src/jsonl.rs`

- [ ] **Step 1: Create the harness crate manifest**

`crates/panoptes-harness/Cargo.toml`:
```toml
[package]
name = "panoptes-harness"
version = "0.1.0"
edition = "2021"

[dependencies]
panoptes-core = { path = "../panoptes-core" }
serde = { workspace = true }
serde_json = { workspace = true }
reqwest = { workspace = true }
tokio = { workspace = true }
async-trait = { workspace = true }
chrono = { workspace = true }
sha2 = { workspace = true }   # response_id hashing in dispatch.rs (Task 9)
clap = { workspace = true }
anyhow = { workspace = true }

[dev-dependencies]
wiremock = "0.6"
tempfile = "3"
```

- [ ] **Step 2: Define the client trait + response type**

`crates/panoptes-harness/src/client.rs`:
```rust
use async_trait::async_trait;
use panoptes_core::Usage;

#[derive(Debug, Clone)]
pub struct ModelResponse {
    pub text: String,
    pub usage: Usage,
}

#[async_trait]
pub trait ModelClient: Send + Sync {
    /// Exact pinned model string, e.g. "anthropic/claude-x".
    fn model_name(&self) -> &str;
    /// Single-turn generation. Clean context: prompt is the entire input.
    async fn generate(&self, prompt: &str) -> anyhow::Result<ModelResponse>;
}
```

- [ ] **Step 3: Write the failing test for the append-only writer**

`crates/panoptes-harness/src/jsonl.rs`:
```rust
use panoptes_core::ResponseRecord;
use std::fs::OpenOptions;
use std::io::Write;
use std::path::Path;

/// Append one record as a single JSON line. Never truncates — the log is sacred.
pub fn append_record(path: &Path, rec: &ResponseRecord) -> anyhow::Result<()> {
    let mut f = OpenOptions::new().create(true).append(true).open(path)?;
    writeln!(f, "{}", serde_json::to_string(rec)?)?;
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use panoptes_core::params::{Params, Reversibility, TimePressure};
    use panoptes_core::Usage;

    fn rec(id: &str) -> ResponseRecord {
        ResponseRecord {
            response_id: id.into(), vignette_id: "v".into(), model: "m".into(), epoch: 0,
            params: Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
                reversibility: Reversibility::Reversible, info_request: true },
            prompt: "p".into(), response: "r".into(),
            usage: Usage { input_tokens: 1, output_tokens: 2 },
            run_at: "2026-07-17T12:00:00Z".parse().unwrap(),
        }
    }

    #[test]
    fn appends_without_truncating() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("responses.jsonl");
        append_record(&path, &rec("a")).unwrap();
        append_record(&path, &rec("b")).unwrap();
        let contents = std::fs::read_to_string(&path).unwrap();
        assert_eq!(contents.lines().count(), 2, "second append must not overwrite the first");
        // each line is independently valid JSON
        for line in contents.lines() {
            serde_json::from_str::<ResponseRecord>(line).unwrap();
        }
    }
}
```

- [ ] **Step 4: Write lib.rs**

`crates/panoptes-harness/src/lib.rs`:
```rust
pub mod client;
pub mod jsonl;
pub use client::{ModelClient, ModelResponse};
pub use jsonl::append_record;
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cargo test -p panoptes-harness jsonl`
Expected: PASS, 1 test. `appends_without_truncating` guards the append-only invariant.

- [ ] **Step 6: Commit**

```bash
git add crates/panoptes-harness
git commit -m "feat(harness): ModelClient trait + append-only JSONL writer"
```

---

## Task 8: Anthropic client against a mock server (Stage 2b)

**Files:**
- Create: `crates/panoptes-harness/src/anthropic.rs`
- Modify: `crates/panoptes-harness/src/lib.rs`

- [ ] **Step 1: Write the failing test using wiremock**

`crates/panoptes-harness/src/anthropic.rs`:
```rust
use crate::client::{ModelClient, ModelResponse};
use async_trait::async_trait;
use panoptes_core::Usage;
use serde::Deserialize;

pub struct AnthropicClient {
    pub api_key: String,
    pub model: String,
    pub base_url: String, // injectable so tests can point at a mock
    http: reqwest::Client,
}

impl AnthropicClient {
    pub fn new(api_key: String, model: String, base_url: String) -> Self {
        Self { api_key, model, base_url, http: reqwest::Client::new() }
    }
}

#[derive(Deserialize)]
struct RawUsage { input_tokens: u32, output_tokens: u32 }
#[derive(Deserialize)]
struct RawBlock { text: String }
#[derive(Deserialize)]
struct RawResp { content: Vec<RawBlock>, usage: RawUsage }

#[async_trait]
impl ModelClient for AnthropicClient {
    fn model_name(&self) -> &str { &self.model }

    async fn generate(&self, prompt: &str) -> anyhow::Result<ModelResponse> {
        let raw: RawResp = self.http
            .post(format!("{}/v1/messages", self.base_url))
            .header("x-api-key", &self.api_key)
            .header("anthropic-version", "2023-06-01")
            .json(&serde_json::json!({
                "model": self.model, "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            }))
            .send().await?
            .error_for_status()?
            .json().await?;
        let text = raw.content.into_iter().map(|b| b.text).collect::<Vec<_>>().join("");
        Ok(ModelResponse { text, usage: Usage {
            input_tokens: raw.usage.input_tokens, output_tokens: raw.usage.output_tokens } })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use wiremock::matchers::{method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    #[tokio::test]
    async fn parses_a_mocked_completion() {
        let server = MockServer::start().await;
        Mock::given(method("POST")).and(path("/v1/messages"))
            .respond_with(ResponseTemplate::new(200).set_body_json(serde_json::json!({
                "content": [{"type": "text", "text": "Recommend SENSOR_RETASK."}],
                "usage": {"input_tokens": 42, "output_tokens": 7}
            })))
            .mount(&server).await;

        let client = AnthropicClient::new("k".into(), "claude-x".into(), server.uri());
        let resp = client.generate("test prompt").await.unwrap();
        assert_eq!(resp.text, "Recommend SENSOR_RETASK.");
        assert_eq!(resp.usage.input_tokens, 42);
        assert_eq!(resp.usage.output_tokens, 7);
    }
}
```

- [ ] **Step 2: Re-export from lib.rs**

Append to `crates/panoptes-harness/src/lib.rs`:
```rust
pub mod anthropic;
pub use anthropic::AnthropicClient;
```

- [ ] **Step 3: Run the test to verify it passes**

Run: `cargo test -p panoptes-harness anthropic`
Expected: PASS, 1 test. The mock proves parsing without spending API credits. Adding OpenAI/other providers = copy this file, change the request/response shapes.

- [ ] **Step 4: Commit**

```bash
git add crates/panoptes-harness
git commit -m "feat(harness): Anthropic client with mock-server test"
```

---

## Task 9: Dispatch loop over manifest × models × epochs (Stage 2c)

**Files:**
- Create: `crates/panoptes-harness/src/dispatch.rs`
- Create: `crates/panoptes-harness/src/main.rs`
- Modify: `crates/panoptes-harness/Cargo.toml` (add `[[bin]]`)
- Modify: `crates/panoptes-harness/src/lib.rs`

- [ ] **Step 1: Write the failing test for opaque response IDs**

`crates/panoptes-harness/src/dispatch.rs`:
```rust
use panoptes_core::{ResponseRecord, Vignette};
use crate::client::ModelClient;
use crate::jsonl::append_record;
use chrono::Utc;
use sha2::{Digest, Sha256};
use std::path::Path;

/// Opaque, deterministic response id — coders must not be able to read the
/// model or parameters off it (blinding). Hash of (vignette_id, model, epoch).
pub fn response_id(vignette_id: &str, model: &str, epoch: u32) -> String {
    let mut h = Sha256::new();
    h.update(vignette_id.as_bytes());
    h.update(model.as_bytes());
    h.update(epoch.to_le_bytes());
    format!("{:x}", h.finalize())[..16].to_string()
}

pub async fn dispatch(
    vignettes: &[Vignette],
    clients: &[Box<dyn ModelClient>],
    epochs: u32,
    log_path: &Path,
) -> anyhow::Result<usize> {
    let mut count = 0;
    for v in vignettes {
        for client in clients {
            for epoch in 0..epochs {
                let resp = client.generate(&v.prompt).await?;
                append_record(log_path, &ResponseRecord {
                    response_id: response_id(&v.id, client.model_name(), epoch),
                    vignette_id: v.id.clone(), model: client.model_name().to_string(),
                    epoch, params: v.params, prompt: v.prompt.clone(),
                    response: resp.text, usage: resp.usage, run_at: Utc::now(),
                })?;
                count += 1;
            }
        }
    }
    Ok(count)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn response_id_is_deterministic_and_opaque() {
        let a = response_id("ca_geo-030-H-REV-INFO", "claude-x", 0);
        let b = response_id("ca_geo-030-H-REV-INFO", "claude-x", 0);
        assert_eq!(a, b, "same inputs → same id");
        assert_ne!(a, response_id("ca_geo-030-H-REV-INFO", "claude-x", 1), "epoch changes id");
        assert!(!a.contains("claude"), "model name must not leak into the id");
        assert!(!a.contains("030"), "params must not leak into the id");
        assert_eq!(a.len(), 16);
    }
}
```

- [ ] **Step 2: Write an integration test for the full loop against a mock**

Append to the `tests` module in `dispatch.rs`:
```rust
    use crate::anthropic::AnthropicClient;
    use panoptes_core::params::{Params, Reversibility, TimePressure};
    use wiremock::matchers::method;
    use wiremock::{Mock, MockServer, ResponseTemplate};

    fn vig(id: &str) -> Vignette {
        Vignette { id: id.into(), family: "ca_geo".into(),
            params: Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
                reversibility: Reversibility::Reversible, info_request: true },
            prompt: "p".into(), prompt_sha256: "x".into() }
    }

    #[tokio::test]
    async fn dispatch_writes_one_record_per_call() {
        let server = MockServer::start().await;
        Mock::given(method("POST")).respond_with(ResponseTemplate::new(200).set_body_json(
            serde_json::json!({"content":[{"type":"text","text":"ok"}],
                "usage":{"input_tokens":1,"output_tokens":1}})))
            .mount(&server).await;

        let clients: Vec<Box<dyn ModelClient>> = vec![
            Box::new(AnthropicClient::new("k".into(), "claude-x".into(), server.uri())),
        ];
        let vignettes = vec![vig("a"), vig("b")];
        let dir = tempfile::tempdir().unwrap();
        let log = dir.path().join("responses.jsonl");

        // 2 vignettes × 1 client × 3 epochs = 6 records
        let n = dispatch(&vignettes, &clients, 3, &log).await.unwrap();
        assert_eq!(n, 6);
        assert_eq!(std::fs::read_to_string(&log).unwrap().lines().count(), 6);
    }
```

- [ ] **Step 3: Write the run binary**

`crates/panoptes-harness/src/main.rs`:
```rust
use anyhow::Result;
use clap::Parser;
use panoptes_core::Vignette;
use panoptes_harness::{dispatch::dispatch, AnthropicClient, ModelClient};
use std::fs;
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// Directory containing manifest.csv + prompts/ (from panoptes-gen)
    #[arg(long, default_value = "scenarios/generated")]
    scenarios: PathBuf,
    /// Append-only log path
    #[arg(long, default_value = "harness/logs/responses.jsonl")]
    log: PathBuf,
    #[arg(long, default_value_t = 5)]
    epochs: u32,
}

fn load_vignettes(dir: &std::path::Path) -> Result<Vec<Vignette>> {
    // Reconstruct vignettes from manifest + prompt files.
    // (Parsing left as a small CSV read; the manifest schema is fixed in Task 6.)
    let manifest = fs::read_to_string(dir.join("manifest.csv"))?;
    let mut out = Vec::new();
    for line in manifest.lines().skip(1) {
        let cols: Vec<&str> = line.split(',').collect();
        let id = cols[0].to_string();
        let prompt = fs::read_to_string(dir.join("prompts").join(format!("{id}.txt")))?;
        out.push(Vignette {
            id: id.clone(), family: cols[1].to_string(),
            params: panoptes_core::params::Params {
                attribution_confidence: cols[2].parse()?,
                time_pressure: cols[3].parse().map_err(|e| anyhow::anyhow!("{e}"))?,
                reversibility: cols[4].parse().map_err(|e| anyhow::anyhow!("{e}"))?,
                info_request: cols[5].parse()?,
            },
            prompt, prompt_sha256: cols[6].to_string(),
        });
    }
    Ok(out)
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    fs::create_dir_all(cli.log.parent().unwrap())?;
    let vignettes = load_vignettes(&cli.scenarios)?;

    // Pinned model list — final list is Open Question #6. base_url from env for real runs.
    let base = std::env::var("ANTHROPIC_BASE_URL").unwrap_or("https://api.anthropic.com".into());
    let key = std::env::var("ANTHROPIC_API_KEY").unwrap_or_default();
    let clients: Vec<Box<dyn ModelClient>> = vec![
        Box::new(AnthropicClient::new(key, "claude-x-pinned".into(), base)),
    ];

    let n = dispatch(&vignettes, &clients, cli.epochs, &cli.log).await?;
    eprintln!("wrote {n} response records → {}", cli.log.display());
    Ok(())
}
```

- [ ] **Step 4: Declare the binary + re-export dispatch**

Append to `crates/panoptes-harness/Cargo.toml`:
```toml
[[bin]]
name = "panoptes-run"
path = "src/main.rs"
```

Append to `crates/panoptes-harness/src/lib.rs`:
```rust
pub mod dispatch;
pub use dispatch::{dispatch, response_id};
```

- [ ] **Step 5: Run all harness tests**

Run: `cargo test -p panoptes-harness`
Expected: PASS — `response_id_is_deterministic_and_opaque` and `dispatch_writes_one_record_per_call` are the critical two.

- [ ] **Step 6: Commit**

```bash
git add crates/panoptes-harness
git commit -m "feat(harness): dispatch loop + run binary over manifest"
```

---

## Task 10: Blinded coding sheets + blind key (Stage 3a)

**Files:**
- Create: `crates/panoptes-coding/Cargo.toml`
- Create: `crates/panoptes-coding/src/lib.rs`
- Create: `crates/panoptes-coding/src/sheets.rs`

- [ ] **Step 1: Create the coding crate manifest**

`crates/panoptes-coding/Cargo.toml`:
```toml
[package]
name = "panoptes-coding"
version = "0.1.0"
edition = "2021"

[dependencies]
panoptes-core = { path = "../panoptes-core" }
serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }   # CodingError derive in validate.rs (Task 11)
clap = { workspace = true }
anyhow = { workspace = true }

[dev-dependencies]
tempfile = "3"
```

- [ ] **Step 2: Write the failing test — sheet must not leak model or params**

`crates/panoptes-coding/src/sheets.rs`:
```rust
use panoptes_core::ResponseRecord;
use std::collections::BTreeMap;

/// A blank coding row: response_id + the response text, and nothing that
/// reveals which model produced it or what the parameters were.
#[derive(Debug, Clone, serde::Serialize)]
pub struct BlankSheetRow {
    pub response_id: String,
    pub response: String,
    // coder fills these; empty in the blank
    pub c1_action: String,
    pub c2_logic: String,
    pub c2_anchor_quote: String,
    pub c3_escalation: String,
    pub codebook_version: String,
    pub coder_notes: String,
}

/// The blind key, kept separate and NOT given to coders.
#[derive(Debug, Clone, serde::Serialize)]
pub struct KeyRow {
    pub response_id: String,
    pub vignette_id: String,
    pub model: String,
    pub epoch: u32,
}

pub fn make_sheet(records: &[ResponseRecord]) -> (Vec<BlankSheetRow>, Vec<KeyRow>) {
    let mut sheet = Vec::new();
    let mut key = Vec::new();
    for r in records {
        sheet.push(BlankSheetRow {
            response_id: r.response_id.clone(), response: r.response.clone(),
            c1_action: String::new(), c2_logic: String::new(),
            c2_anchor_quote: String::new(), c3_escalation: String::new(),
            codebook_version: String::new(), coder_notes: String::new(),
        });
        key.push(KeyRow {
            response_id: r.response_id.clone(), vignette_id: r.vignette_id.clone(),
            model: r.model.clone(), epoch: r.epoch,
        });
    }
    (sheet, key)
}

/// Deterministic shuffle by seed so ordering can't be reconstructed but runs
/// are reproducible. Uses a simple index permutation keyed on the seed.
pub fn shuffled_indices(n: usize, seed: u64) -> Vec<usize> {
    // Fisher–Yates with a tiny LCG — no external rand dependency.
    let mut idx: Vec<usize> = (0..n).collect();
    let mut state = seed.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
    for i in (1..n).rev() {
        state = state.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
        let j = (state >> 33) as usize % (i + 1);
        idx.swap(i, j);
    }
    idx
}

#[cfg(test)]
mod tests {
    use super::*;
    use panoptes_core::params::{Params, Reversibility, TimePressure};
    use panoptes_core::Usage;

    fn rec(id: &str, model: &str) -> ResponseRecord {
        ResponseRecord {
            response_id: id.into(), vignette_id: "ca_geo-030-H-REV-INFO".into(),
            model: model.into(), epoch: 0,
            params: Params { attribution_confidence: 30, time_pressure: TimePressure::Hours,
                reversibility: Reversibility::Reversible, info_request: true },
            prompt: "p".into(), response: "Recommend MONITOR.".into(),
            usage: Usage { input_tokens: 1, output_tokens: 1 },
            run_at: "2026-07-17T12:00:00Z".parse().unwrap(),
        }
    }

    #[test]
    fn sheet_does_not_leak_identity() {
        let (sheet, key) = make_sheet(&[rec("r1", "anthropic/claude-x")]);
        let serialized = serde_json::to_string(&sheet).unwrap();
        assert!(!serialized.contains("claude"), "model must not appear on the sheet");
        assert!(!serialized.contains("ca_geo"), "vignette id must not appear on the sheet");
        assert!(!serialized.contains("30"), "params must not appear on the sheet");
        // but the key retains the mapping
        assert_eq!(key[0].model, "anthropic/claude-x");
        assert_eq!(key[0].vignette_id, "ca_geo-030-H-REV-INFO");
    }

    #[test]
    fn shuffle_is_deterministic_and_a_permutation() {
        let a = shuffled_indices(18, 20261);
        let b = shuffled_indices(18, 20261);
        assert_eq!(a, b, "same seed → same order");
        let mut sorted = a.clone(); sorted.sort();
        assert_eq!(sorted, (0..18).collect::<Vec<_>>(), "must be a permutation of 0..n");
    }
}
```

- [ ] **Step 3: Write lib.rs**

`crates/panoptes-coding/src/lib.rs`:
```rust
pub mod sheets;
pub use sheets::{make_sheet, shuffled_indices, BlankSheetRow, KeyRow};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cargo test -p panoptes-coding sheets`
Expected: PASS, 2 tests. `sheet_does_not_leak_identity` is the integrity guarantee for blind coding.

- [ ] **Step 5: Commit**

```bash
git add crates/panoptes-coding
git commit -m "feat(coding): blinded coding sheets + separate blind key"
```

---

## Task 11: Coded-CSV loader — parse is validation (Stage 3b)

**Files:**
- Create: `crates/panoptes-coding/src/validate.rs`
- Create: `crates/panoptes-coding/src/main.rs`
- Modify: `crates/panoptes-coding/Cargo.toml` (add `[[bin]]`)
- Modify: `crates/panoptes-coding/src/lib.rs`

- [ ] **Step 1: Write the failing test — bad codebook value fails loudly with row context**

`crates/panoptes-coding/src/validate.rs`:
```rust
use panoptes_core::CodedRow;

#[derive(Debug, thiserror::Error)]
#[error("row {line}: {source}")]
pub struct CodingError {
    pub line: usize,
    #[source]
    pub source: serde_json::Error,
}

/// Load coded rows from a JSON-lines file. Each line deserializes into a
/// CodedRow, so any value outside the codebook enums fails here — the
/// validation IS the parse. Empty anchor quotes for latent codes are caught
/// by the check below.
pub fn load_coded(contents: &str) -> Result<Vec<CodedRow>, CodingError> {
    let mut rows = Vec::new();
    for (i, line) in contents.lines().enumerate() {
        if line.trim().is_empty() { continue; }
        let row: CodedRow = serde_json::from_str(line)
            .map_err(|source| CodingError { line: i + 1, source })?;
        rows.push(row);
    }
    Ok(rows)
}

/// Latent codes (C2 strategic logic) must anchor to quoted text — the codebook rule.
pub fn check_anchors(rows: &[CodedRow]) -> Result<(), String> {
    for r in rows {
        if r.c2_anchor_quote.trim().is_empty() {
            return Err(format!("{}: C2 logic coded without an anchor quote", r.response_id));
        }
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    const VALID: &str = r#"{"response_id":"r1","c1_action":"SENSOR_RETASK","c2_logic":"CONTROL","c2_anchor_quote":"positional advantage","c3_escalation":0,"codebook_version":"0.3","coder_notes":""}"#;
    const BAD_ENUM: &str = r#"{"response_id":"r2","c1_action":"SENSOR_RETASK","c2_logic":"AGGRESSIVE","c2_anchor_quote":"x","c3_escalation":0,"codebook_version":"0.3","coder_notes":""}"#;
    const NO_ANCHOR: &str = r#"{"response_id":"r3","c1_action":"MONITOR","c2_logic":"CONTROL","c2_anchor_quote":"","c3_escalation":0,"codebook_version":"0.3","coder_notes":""}"#;

    #[test]
    fn valid_row_loads() {
        let rows = load_coded(VALID).unwrap();
        assert_eq!(rows.len(), 1);
        assert_eq!(rows[0].response_id, "r1");
    }

    #[test]
    fn bad_enum_fails_with_line_number() {
        let err = load_coded(BAD_ENUM).unwrap_err();
        assert_eq!(err.line, 1);
        // proves an out-of-codebook value can't slip through
    }

    #[test]
    fn missing_anchor_is_caught() {
        let rows = load_coded(NO_ANCHOR).unwrap();
        let err = check_anchors(&rows).unwrap_err();
        assert!(err.contains("r3"));
        assert!(err.contains("anchor"));
    }
}
```

- [ ] **Step 2: Write the validation CLI**

`crates/panoptes-coding/src/main.rs`:
```rust
use anyhow::Result;
use clap::Parser;
use panoptes_coding::validate::{check_anchors, load_coded};
use std::fs;
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// Path to a coded JSON-lines file (primary or second coder)
    #[arg(long)]
    coded: PathBuf,
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let contents = fs::read_to_string(&cli.coded)?;
    let rows = load_coded(&contents).map_err(|e| anyhow::anyhow!("{e}"))?;
    check_anchors(&rows).map_err(|e| anyhow::anyhow!(e))?;
    eprintln!("OK: {} coded rows valid against the codebook", rows.len());
    Ok(())
}
```

- [ ] **Step 3: Declare the binary + re-export**

Append to `crates/panoptes-coding/Cargo.toml`:
```toml
[[bin]]
name = "panoptes-code"
path = "src/main.rs"
```

Append to `crates/panoptes-coding/src/lib.rs`:
```rust
pub mod validate;
pub use validate::{check_anchors, load_coded, CodingError};
```

- [ ] **Step 4: Run tests, then exercise the CLI on a good and a bad file**

Run: `cargo test -p panoptes-coding validate`
Expected: PASS, 3 tests.

Create a temp bad file and confirm the CLI rejects it:
```bash
echo '{"response_id":"r2","c1_action":"SENSOR_RETASK","c2_logic":"AGGRESSIVE","c2_anchor_quote":"x","c3_escalation":0,"codebook_version":"0.3","coder_notes":""}' > /tmp/bad.jsonl
cargo run -p panoptes-coding -- --coded /tmp/bad.jsonl; echo "exit: $?"
```
Expected: non-zero exit with an error naming line 1 — the codebook constraint enforced at the boundary.

- [ ] **Step 5: Commit**

```bash
git add crates/panoptes-coding
git commit -m "feat(coding): coded-CSV loader where parsing enforces the codebook"
```

---

## Task 12: Workspace lints, Stage-4 handoff stub, README

**Files:**
- Create: `reliability/README.md` (Python handoff contract — not implemented in Rust)
- Create: `README.md` (workspace overview + run sequence)
- Modify: root `Cargo.toml` (workspace lints)

- [ ] **Step 1: Add workspace-wide lint config**

Append to root `Cargo.toml`:
```toml
[workspace.lints.rust]
unused_must_use = "deny"

[workspace.lints.clippy]
unwrap_used = "warn"     # unwraps are fine in tests; flag them in library code
```

Add to each crate's `Cargo.toml` (below `[package]`):
```toml
[lints]
workspace = true
```

- [ ] **Step 2: Write the Stage-4 handoff contract**

`reliability/README.md`:
```markdown
# Stage 4 — Reliability (Python, not Rust)

Reads:
- `coding/coded/primary.csv`   (validated by `panoptes-code`)
- `coding/coded/second_coder.csv`
- `coding/blind_key.csv`       (to join response_id → vignette_id)

Produces:
- `reliability/irr_report.csv`  — per-criterion Cohen's κ / Krippendorff's α
- `reliability/disagreements_<criterion>.csv`

Why Python: `krippendorff` and `scikit-learn`'s `cohen_kappa_score` are correct,
maintained, and not worth reimplementing. C3 escalation uses ordinal α.
Stratify the reliability subsample by `family` (join via blind_key) so no
scenario type escapes validation. Freeze the codebook version only when every
criterion clears ~0.7 (0.8 comfort). This table is Ch3/Ch4 verbatim.
```

- [ ] **Step 3: Write the workspace README with the run sequence**

`README.md`:
```markdown
# Panoptes Evaluation Harness

Stages 1–4 in Rust (typed, reproducible); Stages 5–6 in Python (stats + analysis),
reading the file contract this workspace produces.

## Run sequence
1. Generate:  `cargo run -p panoptes-gen -- --family scenarios/families/ca_geo.toml`
2. Execute:   `ANTHROPIC_API_KEY=... cargo run -p panoptes-run -- --epochs 5`
3. Sheets:    `cargo run -p panoptes-code -- ...`  (blank sheets → human codes → validate)
4. Validate:  `cargo run -p panoptes-code -- --coded coding/coded/primary.csv`
5–6. Python analysis reads harness/logs/responses.jsonl + coding/coded/*.csv

## Invariants
- responses.jsonl is append-only (the dataset of record)
- coding sheets never contain model identity or parameters (blinding)
- codebook values are enums — invalid values fail to parse, not lint
- the manifest is the join spine: analysis joins on vignette_id / response_id
```

- [ ] **Step 4: Full workspace build + test + clippy**

Run: `cargo build --workspace && cargo test --workspace && cargo clippy --workspace`
Expected: builds clean, all tests pass, clippy warns only on any intentional library `unwrap`s.

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml crates README.md reliability/README.md
git commit -m "chore: workspace lints, Stage-4 handoff contract, README"
```

---

## Self-Review

**Spec coverage.** Every stage from the architecture is implemented: Stage 1 generation (Tasks 4–6), Stage 2 execution (Tasks 7–9), Stage 3 coding (Tasks 10–11), with the shared data model front-loaded (Tasks 1–3) so types are defined once. Stage 4 is deliberately a Python handoff contract (Task 12), matching the decision to keep reliability statistics in Python. The three design invariants from earlier sessions each have a guarding test: append-only (`appends_without_truncating`), blinding (`sheet_does_not_leak_identity`, `response_id_is_deterministic_and_opaque`), and codebook-as-type (`invalid_logic_is_rejected`, `bad_enum_fails_with_line_number`).

**Type consistency.** `Params`, `Vignette`, `ResponseRecord`, `CodedRow`, and the enums are defined once in `panoptes-core` and imported everywhere else. `vignette_id` (core) and `response_id` (harness) are distinct by design: the former is human-readable and encodes parameters; the latter is opaque and hides them, which is the blinding property. `model_name()` on `ModelClient` is the single source of the pinned model string used in both dispatch and records.

**Known simplifications to harden during the build, not blockers.** The manifest CSV is written and parsed by hand (Tasks 6, 9) rather than via a `csv` crate — fine at this scale, but swap to the `csv` crate if quoting/escaping in prompts ever leaks into manifest fields (prompts live in separate files, so this is low-risk). The Anthropic request/response shapes (Task 8) are current-API-shaped but must be checked against live docs at build time. Provider list in `main` (Task 9) is a placeholder pending Open Question #6 (final pinned model list).

