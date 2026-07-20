# The Architecture at a Glance

<style>
/* Render each diagram on a fixed light "plate" so it stays legible on the book's
   dark navy theme (which would otherwise force the SVG text light-on-light). */
.mermaid { background:#f3f5f8; border:1px solid #d6dee7; border-radius:12px; padding:1.1rem 0.6rem; overflow-x:auto; }
.mermaid text, .mermaid tspan, .mermaid span, .mermaid div, .mermaid p, .mermaid .nodeLabel, .mermaid .edgeLabel, .mermaid foreignObject * { fill:#1b2530 !important; color:#1b2530 !important; }
.mermaid .edgeLabel, .mermaid .edgeLabel rect, .mermaid .edgeLabel div, .mermaid .edgeLabel span { background-color:#f3f5f8 !important; }
</style>

How the structs, enums, traits, and functions of the finished Panoptes harness connect. Four crates, built in dependency order — every stage depends only on `panoptes-core`'s types, never on each other:

`panoptes-gen` → `panoptes-harness` → `panoptes-coding`, and all three → `panoptes-core`.

## The pipeline

The full four-stage path, orchestrated by the unified `panoptes` CLI. A TOML spec and a 24-cell parameter grid become vignettes; the dispatch loop runs them across models and epochs into an append-only log; blinded sheets go out for coding and come back as typed `CodedRow`s that Python reads downstream.

```mermaid
%%{init:{'theme':'base','themeVariables':{'fontSize':'13px','lineColor':'#7c8b9a','textColor':'#1b2530','primaryColor':'#e9eef3','primaryBorderColor':'#9fb0bf','primaryTextColor':'#1b2530','clusterBkg':'#fbfcfd','clusterBorder':'#cbd5de'}}}%%
flowchart TB
  cli{{"panoptes CLI · generate · run · code"}}:::cli
  toml[("ca_geo.toml")]:::store --> ft["FamilySpec::from_toml()"]:::gen
  ft --> spec["FamilySpec"]:::gen
  grid["all_params() · 24 Params (3×2×2×2)"]:::gen --> gen
  spec --> gen["generate(CaGeo, spec)"]:::gen
  fam["CaGeo : ScenarioFamily"]:::gen -->|"is_valid() filters the grid"| gen
  gen --> vig["Vignette · id · params · prompt · sha256"]:::core
  vig --> files[("prompts/*.txt · manifest.csv")]:::store
  files --> disp["dispatch() · vignette × model × epoch"]:::harness
  disp --> client["ModelClient::generate() · AnthropicClient"]:::harness
  client --> resp["ModelResponse · text · Usage"]:::harness
  resp --> rec["ResponseRecord · response_id · params · usage"]:::core
  rec --> app["append_record()"]:::harness
  app --> jsonl[("responses.jsonl · append-only")]:::store
  jsonl --> sheets["make_sheet() · shuffled_indices(seed)"]:::coding
  sheets --> blank[("sheets/*.csv · blind_key.csv")]:::store
  blank --> human["human / LLM coding"]:::ext
  human --> csv[("coded/*.csv")]:::store
  csv --> load["load_coded() · check_anchors()"]:::coding
  load --> coded["CodedRow · ActionType · StrategicLogic · escalation"]:::core
  coded --> py[("Python: stats and report")]:::ext
  cli -.-> ft
  cli -.-> disp
  cli -.-> sheets

  classDef core fill:#e2efec,stroke:#1f8f80,color:#12231f;
  classDef gen fill:#e6ecf5,stroke:#4a6fa5,color:#182436;
  classDef harness fill:#f6ead4,stroke:#b9791f,color:#3a2a10;
  classDef coding fill:#efe6f4,stroke:#7d5ba6,color:#2c1f36;
  classDef store fill:#eef1f5,stroke:#6b7a89,color:#1b2530;
  classDef ext fill:#f0f1f3,stroke:#9aa7b3,color:#3a444e,stroke-dasharray:4 3;
  classDef cli fill:#e8ecef,stroke:#4b5560,color:#1b2530,stroke-width:2px;
```

*Teal = `core` · slate = `gen` · amber = `harness` · plum = `coding` · cylinders = files · dashed = outside the Rust workspace · the CLI (dotted) orchestrates the three stages.*

## Type relationships

Composition (`*--` owns-a), trait implementation (`..|>`), and the enums that make invalid states unrepresentable. `Params` is the hub — it is the grid element, it rides inside every `Vignette`, and it is stamped onto every `ResponseRecord`. Two traits define the extension points: `ScenarioFamily` (new scenario) and `ModelClient` (new provider).

```mermaid
%%{init:{'theme':'neutral','themeVariables':{'fontSize':'13px','lineColor':'#7c8b9a'}}}%%
classDiagram
  namespace panoptes_core {
    class TimePressure {
      <<enum>>
      Hours
      Days
    }
    class Reversibility {
      <<enum>>
      Reversible
      Nonreversible
    }
    class Params {
      +u8 attribution_confidence
      +TimePressure time_pressure
      +Reversibility reversibility
      +bool info_request
    }
    class Vignette {
      +String id
      +String family
      +Params params
      +String prompt
      +String prompt_sha256
    }
    class Usage {
      +u32 input_tokens
      +u32 output_tokens
    }
    class ResponseRecord {
      +String response_id
      +String vignette_id
      +String model
      +u32 epoch
      +Params params
      +String prompt
      +String response
      +Usage usage
      +DateTime run_at
    }
    class ActionType {
      <<enum>>
      SensorRetask
      Maneuver
      Monitor
      EscalateToCommand
      RequestData
      NoAction
    }
    class StrategicLogic {
      <<enum>>
      Control
      Maritime
      Political
      Procedural
      None
      Mixed
    }
    class CodedRow {
      +String response_id
      +ActionType c1_action
      +StrategicLogic c2_logic
      +String c2_anchor_quote
      +u8 c3_escalation
      +String codebook_version
      +Option~String~ code_notes
    }
  }
  namespace panoptes_gen {
    class ScenarioFamily {
      <<trait>>
      +name() String
      +is_valid(Params) bool
    }
    class CaGeo {
      +name() String
      +is_valid(Params) bool
    }
    class FamilySpec {
      +String name
      +u32 version
      +String title
      +Vec~String~ doctrine_refs
      +Vec~String~ action_menu
      +String template
      +from_toml(str) FamilySpec
    }
  }
  namespace panoptes_harness {
    class ModelResponse {
      +String text
      +Usage usage
    }
    class ModelClient {
      <<trait>>
      +model_name() str
      +generate(prompt) ModelResponse
    }
    class AnthropicClient {
      +String api_key
      +String model
      +String base_url
      +new(key, model, url) AnthropicClient
    }
  }
  namespace panoptes_coding {
    class BlankSheetRow {
      +String response_id
      +String response
      +String c1_action
      +String c2_logic
      +String c2_anchor_quote
      +String c3_escalation
    }
    class KeyRow {
      +String response_id
      +String vignette_id
      +String model
      +u32 epoch
    }
    class CodingError {
      <<error>>
      +usize line
      +serde_json::Error source
    }
  }
  Params *-- TimePressure
  Params *-- Reversibility
  Vignette *-- Params
  ResponseRecord *-- Params
  ResponseRecord *-- Usage
  ModelResponse *-- Usage
  CodedRow *-- ActionType
  CodedRow *-- StrategicLogic
  CaGeo ..|> ScenarioFamily
  AnthropicClient ..|> ModelClient
```

*`*--` composition · `..|>` implements · `«enum»`/`«trait»`/`«error»` stereotypes · `~T~` = generic parameter.*

## Per-crate inventory

| Crate | Owns | Key types |
|---|---|---|
| **panoptes-core** | The data model (no I/O) — every cross-crate type, defined once | `TimePressure` · `Reversibility` · `ActionType` · `StrategicLogic` (enums); `Params`; `Vignette` + `vignette_id`; `Usage` · `ResponseRecord`; `CodedRow` |
| **panoptes-gen** | Stage 1 — generation | `ScenarioFamily` (trait) · `CaGeo`; `FamilySpec`; `all_params` · `generate`; the `panoptes-gen` binary |
| **panoptes-harness** | Stage 2 — execution | `ModelClient` (trait) · `ModelResponse` · `AnthropicClient`; `dispatch` · `append_record`; the `panoptes-run` binary |
| **panoptes-coding** | Stage 3 — coding (parse = validation) | `BlankSheetRow` · `KeyRow`; `make_sheet` · `shuffled_indices`; `CodingError` · `load_coded` · `check_anchors`; the `panoptes-code` binary |

The correctness payoff: a coder **cannot** record a value outside the codebook, because `load_coded` parses each row straight into the `ActionType` / `StrategicLogic` enums and an off-codebook value fails to deserialize. The full verified code is in the [Answer Key](./appendix-answer-key.md).
