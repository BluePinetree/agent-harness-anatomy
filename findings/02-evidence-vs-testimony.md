# 02 — Evidence and testimony are different things

**Claim.** Numbers written by LLM-authored code are claims, not measurements. A judgement
about whether a run succeeded must rest on artifacts produced by code the LLM cannot
rewrite. Without that separation, verification is not merely difficult — it is impossible
in principle.

**Confidence: high.** Measured across 29 archived runs.

---

## The distinction

| | Written by | Admissible as |
|---|---|---|
| **Testimony** | LLM-authored experiment code | Record it. Never adjudicate on it |
| **Evidence** | Code the LLM is forbidden to rewrite | The only basis for a verdict |

Obvious once stated. The system had no such line, and every reliability problem in
[03](03-silent-success.md) traces back to its absence.

## Why parsing the agent's output cannot be made to work

The result file was written by generated code. So its schema was authored fresh on each
run. Across 29 runs with a non-empty metrics block:

- The **most common metric key name appears twice.**
- Names have no consistent shape: `resnet18__test_top1`, `linear_regression_test_r2`, and
  others with no split label at all.
- Some `metrics` blocks contain `error`, `traceback`, `host`, `cwd`, `timestamp_end` —
  fields that are not metrics.

Any normalisation layer over this is either a hardcoded alias table, which breaks on the
next run, or fuzzy matching, which reintroduces judgement into what is supposed to be a
deterministic check.

**The fix is not a better parser.** A fixed code layer must own the metric namespace and
write to a separate, versioned block. Whatever the LLM chooses to name stays in testimony,
where it is ignored.

## What was recoverable

| Property | Runs where it can be established |
|---|---|
| Configuration the experiment actually received | **1 / 29** |
| Any validation-set metric | **0 / 29** |
| Epoch count | 3 / 29 — and only by deriving it from per-epoch timing arrays |
| Self-reported success | 20 / 29 |

Twenty runs claim success. Almost none can be checked. The most consequential row is the
zero: **the protocol required selecting on validation and evaluating on test once, and no
run recorded a validation metric at all.** The selection rule was unenforceable and
unobserved for the entire life of the project.

## The read-back that should have existed from the start

A single mechanism closes most of this. The fixed layer writes back what it actually
received:

```
result.json
├── metrics          ← generated code. testimony. never adjudicated on
└── verification     ← fixed code only. evidence
    ├── config         the parsed arguments and the actual command line
    ├── splits         a content hash per data partition — overlap is detectable
    ├── access         how many times the test set was evaluated
    ├── epochs         how many epochs actually ran
    └── selection      what the final model was chosen on
```

Injecting a setting and *honouring* it are different events. Only the second one matters,
and only the second one is hard to observe. Writing the effective configuration back into
the result file makes the difference visible at zero cost.

## Tamper-evidence, and the part that was only ever asserted

"The LLM cannot rewrite the fixed layer" is a security claim, and this project never had a
threat model for it. In this system the fixed files existed only as **845 lines of string
literals inside a generator function** — not as files. They could not be linted, tested,
diffed, or hashed. The layer the whole verification argument rested on was the one layer
that could not be inspected.

The cheap remedy: keep them as real files, copy them into the workspace verbatim, record a
hash of each, and have the verifier compare. Then a modified evidence layer is a detected
rule violation rather than an undetected success.

## Transferable

If a number in your pipeline was produced by code a model wrote, you do not have a
measurement of that number — you have the model's report of it. Decide, explicitly and
early, which files in your system a model may never author, and make those files the only
input to any pass/fail decision.
