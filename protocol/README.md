# Pre-registered protocol

[`preregistration.yaml`](preregistration.yaml) fixes, in advance, what will count as a
successful run: task definitions, thresholds, the model-selection rule, the compute budget,
the failure taxonomy, and the declared limitations.

## Scope — read this before citing it

**This protocol governs work from this repository's first commit onward. It does not cover
the archived runs.**

The point of pre-registration is that the criteria were fixed *before* the results were
seen. That claim needs a timestamp a third party recorded — not a date on the author's own
machine, which anyone can set.

In the original project the file was written but **never committed anywhere**. So there is
no third-party record of when the criteria were fixed, and the 271 archived runs cannot be
covered retroactively. They were analysed after the fact, and
[evidence/INDEX.md](../evidence/INDEX.md) presents them as such.

Committing the file here creates the record. From this commit forward the claim holds; for
anything earlier it does not, and pretending otherwise would defeat the purpose of having a
protocol at all.

## What it specifies

| Section | Contents |
|---|---|
| `tasks` | Three task families with datasets and target metrics |
| `protocol.selection` | Validation split, what selection happens on, **one** test evaluation permitted, and what happens on violation |
| `thresholds` | Per-task pass thresholds, fixed before results, with tolerance |
| `budget` | Epoch cap, experiment timeout, stall timeout, cost calibration |
| `failures` | The failure taxonomy used for classification |
| `reproducibility` | Tiers, and what each requires |
| `limitations` | Eight declared limitations, including that **no methodology search is performed** |

## Known defects in this version

Stated because a protocol nobody criticises is not being used.

- **The tolerance is too tight.** Thresholds allow 2.00 pp, chosen from human
  implementation variance. Measured agent run-to-run variance was 16–23 pp. As written, the
  primary metric is pre-determined to zero. This needs revision before the protocol governs
  anything real.
- **Three runs cannot discriminate.** Wilson intervals for 0/3 and 3/3 overlap
  ([findings/06](../findings/06-why-this-stopped.md)). Any comparative claim needs a larger
  n or a different statistic.
- **`test_evaluations_allowed: 1` was unenforceable.** Nothing recorded how many times the
  test set was evaluated, in any of 77 result files. A rule that cannot be observed is not a
  rule, which is what makes the evidence layer in
  [findings/02](../findings/02-evidence-vs-testimony.md) a prerequisite rather than a
  refinement.

## Changes

Amendments are appended, never edited in place. A protocol whose history can be rewritten
provides no more assurance than no protocol.
