# Claude-assisted audit log

A record of one AI-assisted working session on this repository, kept so that the assistance
behind [CASE_STUDY.md](CASE_STUDY.md) and
[APPLICATION_SNIPPETS.md](APPLICATION_SNIPPETS.md) is inspectable rather than asserted.

This log exists because the case study argues that model output is testimony until something
deterministic confirms it. That argument applies to the case study itself.

---

## Session identification

| | |
|---|---|
| **Date** | 2026-09-09 |
| **Tool** | Claude Code (Anthropic's CLI for Claude), desktop app |
| **Model** | Claude Opus 5 (`claude-opus-5`), reported by the session environment |
| **Repositories read** | this repository (`agent-harness-anatomy`, at `707df3d`); the archived `MARS` published clone (26 commits, `eea4b96`); the `MARS` working clone; the run archive `crewai_prototype/legacy_pre_prereg/`; the hashed evidence copy `MARS-evidence-2026-08/` |
| **Write scope** | new files under `campus_phd/`; a factual correction to `SOURCE.md`; and, in a later pass on the user's instruction, a recount note added to `README.md`, `README.ko.md` and `evidence/INDEX.md` (see *Corrections*). `findings/`, `protocol/`, `decisions/`, `log/` and `notes/` were not modified. **Corrected 2026-09-15:** this row previously ended "Nothing was committed, pushed, tagged or released." That was true when written and false when published — the work was committed and pushed to `origin/main`. No tag or release was created. |
| **Experiments run** | none. No pipeline execution, no training, no model call from `MARS` |

The tool and model names above are what the environment reports. Nothing else about the
provenance of this session is claimed.

## What was asked and what was done

1. Read a task brief and summarise the work it specifies.
2. Scan this repository without modifying it, and locate the primary sources behind each of
   seven named topics.
3. Confirm whether the original `MARS` code named by [SOURCE.md](../SOURCE.md) exists
   locally, and verify against it rather than paraphrasing this repository's prose.
4. Re-derive every headline figure before reusing it.
5. Write `campus_phd/CASE_STUDY.md` and `campus_phd/APPLICATION_SNIPPETS.md`.
6. Verify the results: link resolution, number cross-checks, overclaim and misattribution
   greps, secret and privacy scan, `git diff --check`.

## Files inspected

**This repository, read in full:** `README.md`, `README.ko.md`, `SOURCE.md`, `DEVLOG.md`,
`evidence/INDEX.md`, `findings/README.md`, `findings/01`–`findings/06`,
`decisions/README.md`, `notes/README.md`, `log/INDEX.md`,
`log/working/success_signal_reliability.md`, `protocol/README.md`. Read in part:
`protocol/preregistration.yaml` (structure, plus the `llm`, `protocol.selection`,
`thresholds`, `budget`, `limitations` and `change_log` blocks).

**`MARS` source, read directly:** `crewai_prototype/phases/phase1_planning.py`,
`phase2_coding.py`, `phase3_execution.py`, `phase4_writing.py`,
`orchestration/target_gate.py`, `runtime/liveness.py`, `scaffolds/builder.py`,
`core/llm_factory.py`, `config.yaml`; per-directory line counts; the full commit list.

**Run archive, read and recounted:** `legacy_pre_prereg/README.md`; every `result.json`
under `legacy_pre_prereg/` that parses and is under 20 MB; the directory listings of
`outputs/` and `runs/`; the generated `paper.md` and `result.json` of the CIFAR-10 anchor
run; the two California Housing runs compared in the case study;
`MARS-evidence-2026-08/MANIFEST.md`.

## Claims checked

**Confirmed against code or artifacts.**

| Claim | How it was confirmed |
|---|---|
| Seven framework call sites, each one agent and one task | Read all seven in `phases/` |
| The code-generation phase does not import the framework | No `crewai` import in `phase2_coding.py` (1,209 lines) |
| Layer line counts 2,350 / 3,755 / 1,018 / 845 | Counted per directory; all four matched exactly |
| Completed implementation ≈ 14,500 lines | Measured 14,613 in `crewai_prototype/` |
| 96 `result.json` files; `v3_*` 91; `run_*` 122 | Recounted in the archive |
| A tuning sweep returned the untuned result | Compared the two files: both `0.7756446042829697` |
| A goal check can pass below the goal | Traced the fraction/percent branch in `target_gate.py` by reading it |
| A tree-kill whose non-Windows branch contradicts its docstring | Read `phase3_execution.py:294-315` |
| A liveness helper's "undetermined" read as "alive" by its caller | Read `liveness.py:74-87` against `:203-204` |
| The framework-comparison claim was withdrawn | `MARS` commit `eea4b96` |

**Derived in this session, not previously stated in this form.**

- The CIFAR-10 anchor run's generated paper asserts a 3-epoch budget and cites
  `contract_check.planned_epochs: 3`, a field absent from the `result.json` it was written
  from. That file records `avg_epoch_time_s` equal to `total_train_time_s` for both models,
  which is consistent only with one epoch, while reporting `execution_success: true` and
  `validation_tier: "full"`.
- Across the 44 archived runs with a non-empty `metrics` block there are 306 distinct metric
  key names; the most frequent genuine metric name occurs three times, and the only keys
  recurring more often are wrapper keys from a self-nesting bug.

**Could not be reproduced, or had no single answer.** Recorded rather than worked around.

| Published figure | Recount |
|---|---|
| 271 run directories | **definition-dependent, not simply wrong.** 271 `outputs/` entries, **270** of them directories (the extra one a 1,013-byte `grep.exe.stackdump`), **224** directories containing any file, **276** ids with content across `outputs/` and `runs/`. "Run directories" was never a count of runs |
| The 29-run measurement table (1/29, 0/29, 3/29, 20/29) | 44 runs carry a non-empty metrics block; no filter tried yields 29 |
| 16 of 16 failed experiments produced a paper | 37 runs hold a `paper.md` — reproduced. **The 18 published here was itself not reproducible** (second pass, 2026-09-21): the runs' own status fields say 0 failed, three evidence-based definitions say 15, 13 and 20, and no definition tried yields 18. Withdrawn rather than restated |
| 77 result files | 96 `result.json`, 342 `result*.json` |
| `SOURCE.md`: 13 commits to 2026-07-27 | 26 commits, 2026-06-04 to 2026-08-25 — **corrected in `SOURCE.md` on the user's instruction** |

The scripts that produced the published figures are in neither repository, so these
differences were disclosed rather than adjudicated — first in [CASE_STUDY.md](CASE_STUDY.md)
§7, then in a dated recount note at the front of `README.md`, `README.ko.md` and
`evidence/INDEX.md`. The original figures were not edited to match; they remain in place
there and in `findings/` and `protocol/`.

**Claude-specific provenance, checked because the case study depends on getting it right.**

- Every model string recorded in the run archive is an OpenAI one — `gpt-5.2`, `gpt-5-mini`,
  `gpt-4o-mini`. No Claude model appears. `config.yaml` assigns `provider: openai` to all
  roles, and `protocol/preregistration.yaml` pins `model_all_roles: gpt-5.2`.
- The published `MARS` history carries no `Co-Authored-By` trailer on any of its 26 commits.
- **Corrected 2026-09-15.** This previously read "Both commits in this repository carry
  `Co-Authored-By: Claude Opus 5`." That was true when the repository had two commits and
  false by the time this sentence was published. Exactly two commits carry the trailer
  (`c041467`, `707df3d`); every commit after them does **not**, including the three that
  delivered this case study (`24d46a6`, `cadc45f`, `dac77e9`) and this correction itself. No
  total is given here — a total goes stale on the next commit, which is how the sentence it
  replaces became false. Verify with
  `git log --format='%h %s' --grep='Co-Authored-By: Claude'`.
- The original working tree contained a `CLAUDE.md` and a near-identical `AGENTS.md`, so at
  least two agent harnesses were configured for development. Neither file establishes that
  any particular change was made with either.

Conclusion drawn from that: the archived runs were not Claude runs, and no document produced
in this session says or implies they were.

## Human decisions required

The following were not the assistant's to make, and were referred back:

1. **Whether to work in this repository at all**, rather than in the repository where the
   task brief was found. Decided by the user.
2. **How to handle the five irreproducible figures** — write only re-derived numbers, or
   first attempt to reconstruct the original counting criteria. The user chose re-derived
   numbers only.
3. **Whether to correct `evidence/INDEX.md` and the READMEs, or let the old and new figures
   coexist.** The user first chose plain coexistence, then chose to disclose the recount at
   the front of the repository while leaving the original figures in place. Both passes are
   recorded here.
4. **Whether the case study is accurate about the division of labour.** Asserted from
   [README.md](../README.md) and the commit trailers, and left for the user to confirm.
5. **Whether to change the taglines, add the `evidence/INDEX.md` pointer, commit, or
   publish.** The first two were approved and done. **Corrected 2026-09-15:** this sentence
   previously ended "nothing has been committed or pushed"; the work was in fact committed
   and pushed as `24d46a6`, `cadc45f`, `dac77e9`.

## Corrections made during verification

Two overstatements were caught by the verification pass and fixed before the documents were
finished, recorded here because a verification step that never catches anything has not been
shown to work.

- A draft cited an internal review document as an evidence path. That file is untracked in
  both repositories and `log/INDEX.md` lists it among material deliberately held back. The
  citation was removed; the claim it supported was re-derived from two archived result files
  instead.
- A draft wrote that the epoch arithmetic "proves" one epoch ran, and described the five
  approval gates without noting that the headless harness auto-approved plans. Both were
  corrected — the first to "consistent only with one", the second by adding the caveat that
  `decisions/README.md` records for ADR-005.

A later pass, again on the user's instruction, disclosed the recount at the front of the
repository rather than only inside `campus_phd/`. A dated recount note was added to
[README.md](../README.md), [README.ko.md](../README.ko.md) and
[evidence/INDEX.md](../evidence/INDEX.md); the original figures were left in place there and
in `findings/` and `protocol/`; and both taglines dropped their run count, because
re-measuring showed "run directories" was never a count of runs — 271 entries, 270
directories, 224 non-empty, and 276 ids with content across `outputs/` and `runs/` are all
defensible readings, one of them higher than the published figure. The two counts that had
reproduced exactly (96 result files, and the 122 / 91 generation split) are named as such, so
the note does not imply every figure is in doubt.

One existing document was corrected, on the user's explicit instruction after the discrepancy
was reported. [SOURCE.md](../SOURCE.md) stated "13 commits, 2026-06-04 to 2026-07-27" for the
published repository, and that the published line was complete through 2026-07-27 with later
local work unpushed. Measured: the published `BluePinetree/MARS` branch is **26 commits,
2026-06-04 to 2026-08-25**, and the divergent local branch is **9 commits ending 2026-08-06**
with no common ancestor — so the unpushed work is not *later* than the published tip, it is a
separate lineage. Both statements were updated to the measured values, and the unpushed work
named concretely (`rsp/`, tracked in the local clone and absent from the published tree). No
other existing file was touched; the README's counts were left to coexist with the recount, as
decided.

## Independently verified outputs of this session

Checks that do not depend on the assistant's judgement, with what they returned:

| Check | Result |
|---|---|
| Every relative link in both new documents resolves | 18 links, 0 broken |
| Every `MARS` and archive path cited exists | 11 paths, all present |
| Overclaim vocabulary grep (`first`, `novel`, `state-of-the-art`, `proved`, `fully verified`, …) | No substantive hit remaining |
| Claude-to-historical-run misattribution grep | No hit other than the explicit negative statement |
| Secret, credential, absolute-path and personal-data scan | No hit |
| Word counts against the brief | 49 / 100 / 153; case-study body 1,741 |
| `git diff --check` | Clean |
| `git status` | One untracked directory, `campus_phd/`. No tracked file modified |

## Limits of this audit

It covers one session on one machine. It records what was read and what was checked, not
everything that was considered. It is a log written by the assistant that did the work, so it
is testimony about a process — the parts of it that are evidence are the commands in the
table above, each of which can be re-run against this repository and the archive.
