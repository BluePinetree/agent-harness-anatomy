# 04 — Every failed experiment still produced a paper

**Claim.** Sixteen runs whose experiment stage failed produced a complete generated paper.
Sixteen out of sixteen. The writing stage had no dependency on the execution stage
succeeding.

**Confidence: high.** Complete enumeration, not a sample.

---

## What happened

The pipeline ran `plan → code → execute → analyze → write`. When execution failed, the
pipeline did not stop. It carried the failure forward as a handoff payload like any other,
and the writing stage did what it was built to do: produce a paper from whatever it was
given.

The result is a document with a methods section, a results section, and a conclusion,
describing an experiment that did not produce results.

**16 / 16.** Not a rate — every single case.

## Why this is not a bug in the writing stage

The writing stage worked exactly as specified. It received a handoff, it wrote a paper.
The defect is that **no stage was responsible for asking whether the previous stage had
produced anything worth writing about.**

Each stage trusted its input because its input was a report, and a report is all it got.
That is the same mechanism as [03](03-silent-success.md), one level up: local success
signals propagating into a global claim that nothing checked.

The general shape:

```
execute (failed)  →  "here is my handoff"  →  analyze  →  write  →  paper
                          ^
                          nothing here asks: did the experiment actually succeed?
```

## The part that makes it a research-integrity problem

A generated paper is a publishable-looking artifact. Sixteen of them existed in the archive
alongside papers from successful runs, in the same directory layout, with the same file
names. **Nothing in the artifact distinguishes them.** The only way to tell was to
cross-reference execution status, which lived in a different file.

If any of those sixteen had been read without that cross-reference — by a collaborator, by
a reviewer, by me six weeks later — it would have read as a finding.

This is the concrete version of a documented phenomenon: agent-generated research output
containing claims not supported by the underlying run. Larger studies put fabricated or
invalidated results at a substantial fraction of agent-produced papers. This project
reproduces it at 100% within its own failure set, for a mundane reason: no gate.

## The fix, and why it is not just a gate

The obvious remedy is a gate — do not write if execution failed. Necessary, and it was
added.

But a gate alone is insufficient, because *"did execution succeed"* was itself
unreliable ([02](02-evidence-vs-testimony.md), [03](03-silent-success.md)). Gating on a
self-reported success flag reproduces the problem one step earlier: the phantom search
reported success, so a gate reading that flag would have let it through.

So the gate has to read evidence, not testimony:

| Gate reads | Outcome |
|---|---|
| Self-reported success flag | Reproduces the problem — the flag is testimony |
| Fixed-layer evidence block | Actually gates |

And when the gate blocks, the right behaviour is not silence. The artifact should exist and
say what happened — *execution failed, here is the reason* — because a missing paper is
indistinguishable from a pipeline that never ran.

## Transferable

In a multi-stage pipeline where each stage consumes the previous stage's report, add one
question to every stage: *what evidence, not what claim, tells me the previous stage
succeeded?* If the answer is "its own status field," the pipeline can generate a confident
final artifact from a failed beginning — and it will, every time.
