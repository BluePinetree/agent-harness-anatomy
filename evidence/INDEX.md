# Archived runs

> **Recount note — 2026-09-09.** The counts on this page were re-derived later. The 96
> result files and the 122 / 91 generation split reproduced exactly; the 271 figure turns
> out to depend on an unstated definition (271 `outputs/` entries, 270 of them directories,
> 224 with any file, 276 across `outputs/` and `runs/`); and the 29-run table below could
> not be reproduced — 44 directories carry a non-empty metrics block, and no filter tried
> yields 29. The page is left as written, because it is the record of what was counted at
> the time. See [../README.md](../README.md) for the summary and
> [../campus_phd/CASE_STUDY.md](../campus_phd/CASE_STUDY.md) §7 for the detail.

## Inventory

| | Count |
|---|---|
| Run directories | **271** |
| — `run_<timestamp>_*` generation | 122 |
| — `v3_*` generation | 91 |
| — other / empty | 58 |
| Result files (`result.json`) | **96** |

The archive on disk is 32 GB, of which roughly **250 MB is actual evidence**. The rest is a
dataset re-downloaded once per run (~19.7 GB), model checkpoints, and four pathological
files totalling ~12 GB — including a 4.9 GB `result.json` of 173 million lines caused by a
recursive self-nesting bug.

**That file cannot be opened**; parsing it needs more memory than the machine has. Any
script that walks this archive has to skip it explicitly, which is itself a small lesson
about generated artifacts.

A second lesson: the archive's own README described 122 real directories and 148 empty
ones. Counting them found 271 directories across three naming generations, with 96 result
files — and the 91-directory `v3_*` generation was not mentioned at all. Trusting that
README while cleaning up would have deleted 41 result files.

## What is measurable from these runs

| Property | Runs where it is recoverable |
|---|---|
| Configuration actually received by the experiment | 1 / 29 |
| Validation-set metric | **0 / 29** |
| Epoch count (derived from per-epoch timing arrays) | 3 / 29 |
| Self-reported success | 20 / 29 |

**This table is the point.** Twenty runs claim success; almost none of them can be checked.
That gap is what [findings/02](../findings/02-evidence-vs-testimony.md) is about, and it is
why building the instrumentation became a separate project rather than a patch to this one.

## Availability

The archive is not published here — 32 GB, with dataset redistribution questions attached,
for a retired project. The distilled ~250 MB text subset (generated code, result files,
event streams, handoff state, generated papers) is available on request.
