# Where This Plugs Into the Thesis

> **Kind:** Wrap-up.

You have built the harness. Here is how it sits inside the larger thesis program, so the code you wrote connects to the deadlines it serves.

## The harness is Ch3 made real

Chapter 3 (Methodology and system design) promises an evaluation protocol: parameterized scenarios, pinned models, replication, exhaustive logging, human coding, a reliability plan. Every one of those promises now has running code behind it. When Ch3 says "responses were logged with full prompt, raw output, model version, and parameters," it can cite the `ResponseRecord` schema and the append-only log as *implemented*, not aspirational. The pilot runs in the September window use this harness; Ch3 describes what you actually built.

<div class="callout">
<span class="callout-label">Why the whole apparatus exists</span>
Chip Huyen's <em>AI Engineering</em> names the problem this harness answers directly: because evaluation is difficult, <cite index="0-1">many people settle for word of mouth or eyeballing the results — which creates even more risk and slows iteration; instead, we need to invest in systematic evaluation to make the results more reliable.</cite> Panoptes is that investment for the SDA-decision case. The parameterized scenarios, the frozen codebook, the reliability step, and the reproducible log are what separate a defensible measurement from "the model seemed to do well." That distinction is the thesis's contribution and, later, Norion's product.
</div>

## The four invariants are defense armor

Each correctness property maps to a question a committee can ask:

- **Enums-as-validation** → "How do you know your coded data conforms to the codebook?" Because non-conforming values cannot be parsed.
- **Blinding** → "How do you know your codes are not shaped by knowing which model produced each response?" Because the coding sheet structurally cannot show that.
- **Append-only log** → "How is this reproducible?" Because the dataset of record only ever grows and is archived.
- **Manifest-as-join-spine** → "How do you connect responses to conditions?" Through one deterministic join key.

## The open questions this closed

Recall the Jul–Aug decision list. Building this settles two: the harness is a **ladder build** (you built it deliberately, test-first, and it is portfolio-grade), and the **language question** — Rust for the harness, Python for the stats tail — is resolved at the file contract.

## And the business thread

The same harness is the seed of the evaluation work for the BM3ci TAP Lab subcontract. The scenario-generation and scoring machinery you built for the thesis is the starting point for evaluating a client's space-adapted model — the thesis product and the first contract's tooling share a spine. Building it well here is building it well for both.

<div class="callout trap">
<span class="callout-label">The standing reminder</span>
The harness is done, but the calendar still rules. The September–November window carries Ch2, the pilot, Ch3, and the subcontract at once. Protect the December gate. Do not let harness polish — or a new feature idea that serves no research question — reopen finished work.
</div>
