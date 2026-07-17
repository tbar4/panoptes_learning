# The Python Tail: Stages 5–6

> **Kind:** Wrap-up / pointer. Not built in this course.

Stages 5 (analysis) and 6 (reporting) are Python, by the deliberate boundary decision. This chapter is a map, not an implementation — a future course, or a future set of chapters, could build it out.

## What Stage 4–6 reads

Everything the harness produced:

- `harness/logs/responses.jsonl` — the dataset of record.
- `scenarios/generated/manifest.csv` — the join spine (vignette_id → parameters).
- `coding/coded/primary.csv` and `second_coder.csv` — validated coded data (passed `panoptes-code`).
- `coding/blind_key.csv` — to join response_id → vignette_id after coding is done.

## Stage 4 — Reliability

Compute Cohen's kappa / Krippendorff's alpha per criterion, stratifying the subsample by family (join through the blind key) so no scenario type escapes validation. Dump per-criterion disagreements — those drive codebook revision. Freeze the codebook version only when every criterion clears threshold. This table is Ch3/Ch4 verbatim.

## Stage 5 — Analysis

Join logs to the manifest and to coded data. Ask the designed questions: strategic-logic distribution by model; escalation level as a function of attribution confidence; info-request behavior with and without the option; consistency across replications. The right-for-wrong-reasons detection lives here: a model whose escalation does not move as attribution confidence moves was never conditioning on attribution.

## Stage 6 — Reporting

Tables, figures, and the archived repo (prompts, logs, codebook, coded data, analysis code, all versioned). For the thesis this is Ch4; for the benchmark it is the public release; for the business it becomes the client deliverable. Same stage, three costumes.

<div class="callout">
<span class="callout-label">Why this is a pointer, not a chapter set</span>
The pandas-and-stats work is well-trodden and low-risk relative to the typed harness. It is exactly the kind of thing where the ecosystem is mature and the learning value per hour is lower. If you want, we build it as a short follow-on — but it was never the part that justified a course.
</div>
