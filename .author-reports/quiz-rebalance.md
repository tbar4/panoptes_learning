# Quiz MultipleChoice Length-Tell Rebalance

## Summary

Every `MultipleChoice` question across `quizzes/*.toml` was rewritten so the correct
answer is no longer identifiable as the longest / most-detailed option. Only
`answer.answer` and `prompt.distractors` strings were changed. `prompt.prompt`,
`context`, `id`, and all `ShortAnswer` / `Tracing` questions were left untouched.

- **46 MC questions rewritten** (all of them; 20 files).
- **Length-tell count (answer strictly the longest option): 0 / 46** (was 45 / 46).
- **`mdbook build` exit code: 0** — TOML and Tracing questions still compile.

The "length-tell" metric parses each TOML file and flags a question when
`len(answer.answer) > max(len(distractor))`. Every question now has at least one
distractor as long as or longer than the correct answer, and options were kept
within a tight length band (typical spread under ~15%), with all distractors
rewritten as specific, confident statements of common misconceptions.

## Per-file breakdown

| File | MC rewritten | Final length-tells |
|------|--------------|--------------------|
| 00-orientation-borrowing.toml | 2 | 0 |
| 00-orientation-ownership.toml | 1 | 0 |
| 00-orientation-phase-0-setup.toml | 1 | 0 |
| 01-data-model-concept-enums.toml | 2 | 0 |
| 01-data-model-concept-serde.toml | 4 | 0 |
| 01-data-model-quiz.toml | 3 | 0 |
| 02-generation-concept-cli-files.toml | 2 | 0 |
| 02-generation-concept-iterators.toml | 2 | 0 |
| 02-generation-concept-traits.toml | 2 | 0 |
| 02-generation-quiz.toml | 2 | 0 |
| 03-async-concept-async.toml | 3 | 0 |
| 03-async-concept-mocking.toml | 2 | 0 |
| 03-async-concept-traits-async.toml | 2 | 0 |
| 03-async-quiz.toml | 3 | 0 |
| 04-coding-concept-blinding.toml | 3 | 0 |
| 04-coding-quiz.toml | 2 | 0 |
| 05-hardening-concept-hygiene.toml | 2 | 0 |
| 05-hardening-quiz.toml | 2 | 0 |
| 07-cli-concept-cli.toml | 3 | 0 |
| 07-cli-quiz.toml | 3 | 0 |
| **Total** | **46** | **0** |

## Notes

- The correctness of each answer was preserved; only phrasing/length was rebalanced.
  Claims were kept consistent with the shipped `panoptes` code (e.g. `#[async_trait]`
  usage in `crates/panoptes-harness/src/client.rs`, injectable base URL for mock
  testing, response_id opacity for blinding).
- Question `398e02db` (`#[async_trait]`) uses the exact answer string given as the
  task's worked example; its three distractors were lengthened slightly so the
  answer is not the longest option there either, while keeping the same misconceptions.
- Distractors avoid vague/joke phrasing — each is a plausible, specific misconception
  (e.g. "serde ships a separate derive per format", "the multi-threaded runtime moves
  futures so clients must be Send + Sync only for tests", "clap only accepts enums").
