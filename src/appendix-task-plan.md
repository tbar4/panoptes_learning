# Appendix: The Full Task Plan

This course is paired with a complete, worked implementation plan — the twelve-task, test-first specification that contains the full version of every build chapter.

**Use it as an answer key, not a script.** Attempt each build chapter yourself first, from the behavior described. Then compare against the corresponding task. The gap between your version and the plan is where the learning is.

## Task-to-chapter map

| Task | Chapter |
|---|---|
| 1 — Workspace + parameter types | Part II · Build: Workspace + Parameter Types |
| 2 — Codebook types | Part II · Build: The Codebook Types |
| 3 — Records, vignettes, file contract | Part II · Build: Records, Vignettes, the File Contract |
| 4 — Family spec + validity trait | Part III · Build: Family Spec + Validity Trait |
| 5 — Vignette generation | Part III · Build: Vignette Generation |
| 6 — Generation CLI + manifest | Part III · Build: The Generation CLI + Manifest |
| 7 — ModelClient + append-only log | Part IV · Build: ModelClient + the Append-Only Log |
| 8 — Anthropic client | Part IV · Build: The Anthropic Client |
| 9 — Dispatch loop + run binary | Part IV · Build: The Dispatch Loop + Run Binary |
| 10 — Blinded coding sheets | Part V · Build: Blinded Coding Sheets |
| 11 — Coded-CSV loader | Part V · Build: The Coded-CSV Loader |
| 12 — Lints, handoff, README | Part VI · Build: Lints, Handoff Contract, README |
| 13 — Unified `panoptes` CLI | Part VII · Build: The Unified panoptes Command |

The plan document itself is included in this book: **[Appendix: The Answer Key](./appendix-answer-key.html)** — every manifest, type, attribute, test, and expected output in full.

## The four invariants, in one place

1. **Append-only log** — `responses.jsonl` only ever grows; it is the dataset of record.
2. **Blinding** — coding sheets structurally cannot show model or parameters.
3. **Enums-as-validation** — coded values outside the codebook cannot be parsed.
4. **Manifest-as-join-spine** — one deterministic key ties responses to conditions.

Everything you build should preserve these. If a change would break one, that is the signal to stop and reconsider.
