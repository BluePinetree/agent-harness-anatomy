# From Agent Testimony to Scientific Evidence

**Lessons from an archive of several hundred research-pipeline runs**

> **What this document is.** A retrospective engineering and research record, written for
> readers interested in how a PhD researcher can use AI assistants in scientific work
> without confusing what a model *reports* with what can be *independently checked*.
>
> **Provenance, stated up front.** The system analysed here (`MARS`, see
> [SOURCE.md](../SOURCE.md)) was driven by OpenAI `gpt-5.2`, not by Claude — every model
> string recorded anywhere in its archive is an OpenAI one. Nothing in this document
> claims otherwise. Where Claude appears, it is either the *subject* of reading notes
> ([notes/claude-code-analysis/](../notes/claude-code-analysis/)) or a co-author of this
> repository's own write-up. Section 5 is explicit about which is which.
>
> Every figure below was re-derived from the archive and the source tree while writing
> this document. Where the repository's own published counts differ from that recount,
> section 7 says so rather than smoothing it over.

---

## 1. Why this case study exists

An autonomous research pipeline produces artifacts that look exactly like scientific
output: a result file with numbers in it, and a paper with a methods section and a
conclusion. Neither artifact carries any marking that separates a measurement from a
claim. If the code that wrote the number was itself written by a language model, then the
number is the model's report of the measurement, not the measurement.

That sentence sounds pedantic until you watch it cost you four months. This repository is
the record of watching that. It is **not a controlled benchmark**: one operator, one
machine, one model family, conditions changing as the system was debugged. The pathologies
in it are existence proofs, not rates. What generalises is not a measurement but a
distinction, and the discipline that follows from taking it seriously.

## 2. What I built

A pipeline that took a research question in natural language and produced a paper:
`question → plan → design → code → execute → analyze → write`. Built on CrewAI, with a
React streaming UI and five human-approval gates — which held up in design, though the
headless harness auto-approved plans, so the archived runs record approvals that were
machine-generated ([decisions/README.md](../decisions/README.md), ADR-005). The completed
CrewAI implementation is
**14,613 lines of Python** (`MARS/crewai_prototype/`, measured).

It ran. The archive holds **270 directories** under `outputs/` — **224** of them holding
any file — and **96 `result.json` files**
([evidence/INDEX.md](../evidence/INDEX.md)), and papers were generated end to end.

What was *finished*: the CrewAI pipeline, the long-horizon execution machinery, the UI, the
event stream. What was **not**: AutoGen and LangGraph were partially ported and never ran
end to end, and the improvement loop was off for all three tasks.

**The framework-comparison claim was withdrawn, and that withdrawal stands.** The study was
designed to compare three agent frameworks with all other variables held constant. Counting
the framework's call sites showed the independent variable was never instantiated: `Crew(...)`
appears at **seven** call sites in `MARS/crewai_prototype/phases/`, and every one is
`Crew(agents=[task.agent], tasks=[task])` — one agent, one task. No sequential process, no
hierarchical delegation, no agent-to-agent messaging. The largest phase,
`phase2_coding.py` at 1,209 lines, does not import the framework at all. Three
implementations of *calling the model in a fixed order* are the same program with different
import statements. The comparison could not have produced a difference to measure
([findings/06](../findings/06-why-this-stopped.md)).

I record this because withdrawing a claim on structural grounds is a research skill, and
because the negative result is more useful than the comparison would have been.

## 3. Failures that looked successful

Three worked examples, each re-derived from the archive for this document.

**A tuning sweep that never executed.** One run declared a `GridSearchCV` over 324
configurations with 5-fold cross-validation — **1,620 fits**. Its
gradient-boosting test R² is `0.7756446042829697`. The untuned anchor run's
gradient-boosting test R² is `0.7756446042829697` — identical to all sixteen significant
digits. The configuration had no path to the experiment: the generated `experiment_impl.py`
read `getattr(args, "experiment_config", None)` against a CLI that defines no such
argument, so it was always `None`. The pipeline recorded the run as verified.
*Evidence:* `legacy_pre_prereg/outputs/*_dce015/…/result.json` and `*_c0dad6/…/result.json`
(the two R² values compared directly); the unreachable-argument diagnosis is recorded in an
internal review document that [log/INDEX.md](../log/INDEX.md) lists among the material held
back from publication, so the claim above rests on the two result files alone.

Note the incidental detail: the two runs name the same quantity
`gradient_boosting/test/r2` and `gradient_boosting_test_r2`. The schema was authored fresh
each time.

**A paper asserting an epoch count its own result file refutes.** The CIFAR-10 anchor run's
generated paper states "3 epochs" more than a dozen times and cites a field
`contract_check.planned_epochs: 3`. That field does not exist in the `result.json` the
paper was written from. What the file does contain is
`resnet18.avg_epoch_time_s = 50.30425480008125` and
`resnet18.total_train_time_s = 50.30425480008125` — equal, for both models. If three epochs
had run, the average would be a third of the total. One epoch ran. Meanwhile the same file
reports `execution_success: true`, `validation_tier: "full"`, `evaluation_scope:
"full_test"`.
*Evidence:* `legacy_pre_prereg/outputs/run_20260730_160553_*_d359ba/workspace/results/result.json`
and `…/paper/paper.md`.

**A goal check that passed below the goal.** `MARS/crewai_prototype/orchestration/target_gate.py`
matches metric scale to target scale with `tgt = matched["value_frac"] if value <= 1.0 else
matched["value"]`. The branch assumes the target is the percentage. When the plan states the
target as a fraction (`0.7`) and the generated code reports the metric as a percentage
(`68.86`), the comparison becomes `68.86 >= 0.7` — met — and early-stopping fires below the
goal. The inline comment describes the opposite case, correctly, and the code handles only
that one.

The shape these share: exit code 0, pipeline status complete, self-reported success, and an
outcome that is wrong and undetectable from the artifacts. Four further instances are
catalogued in [findings/03](../findings/03-silent-success.md) — seven in total, of which two
were found *after* the list had been declared complete.

## 4. Evidence vs testimony

The distinction that survived the project:

| | Written by | Admissible as |
|---|---|---|
| **Testimony** | LLM-authored experiment code | Record it. Never adjudicate on it |
| **Evidence** | Code the LLM is forbidden to rewrite | The only basis for a verdict |

Why a better parser cannot substitute. Across the **44 run directories** whose `result.json`
carries a non-empty `metrics` block, there are **306 distinct metric key names**. The most
frequent real metric name occurs **three times**; the only names that recur more often are
wrapper keys (`metrics`, `metrics_unit`, `normalized_metrics`, twelve each) produced by a
self-nesting bug. Some `metrics` blocks hold `error`, `traceback`, `host`, `cwd`. Any
normalisation layer over this is a hardcoded alias table that breaks on the next run, or
fuzzy matching that puts judgement back inside a deterministic check.

Two properties of that same set of 44 make the point sharper. **Twenty-three of them
self-report success.** Exactly **one** carries a metric key naming a validation split — and
that one belongs to the 2026-03 development era the archive's own README marks as
uncitable. The protocol required selecting on validation and evaluating on test once; for
practical purposes no run recorded a validation metric at all, so the selection rule was
unobservable for the life of the project.

The structural fix is cheap and was written into the pre-registration rather than left as
prose: [protocol/preregistration.yaml](../protocol/preregistration.yaml) sets
`success_determined_by: harness` and lists `execution_success`, `validation_tier`,
`dataset_origin`, `evaluation_scope` under
`self_reported_fields_are_recorded_but_not_trusted`. Those four fields are precisely the
ones the CIFAR-10 run used to declare a full-validation success while running one epoch.

One honest caveat about the fixed layer: in this system it existed as **845 lines of string
literals inside a generator function** (`MARS/crewai_prototype/scaffolds/builder.py`), not
as files. It could not be linted, diffed, or hashed. The layer the whole verification
argument rested on was the one layer nobody could inspect — a claim, not a control
([findings/02](../findings/02-evidence-vs-testimony.md)).

## 5. What this changes about how I use Claude

*Stated as current and future practice. The archived runs were `gpt-5.2` runs and I do not
retrofit them.*

What I now bring an assistant into, because none of it produces the numbers a verdict rests
on: **design critique** (asking what would make a planned comparison unanswerable before
running it — the check that would have ended the three-framework design in an afternoon);
**code navigation and seam-reading** (one of the seven silent-success defects lives in the
seam between two individually correct functions, where a reviewer reading either file alone
will not find it, and another in a branch no run ever exercised); **hypothesis generation**
about failure mechanisms, treated as
something to confirm in code; **documentation and translation**; and **failure analysis**
over logs I retain the ability to re-read.

What stays outside, as independently verified: **metric truth**, **dataset splits and their
integrity**, **trial and epoch counts**, **stopping criteria**, and **statistical
verdicts**. These come from a fixed layer the model may not author, ideally content-hashed
so that a modified evidence layer is a detected rule violation rather than an undetected
success.

This division is why some commits in this repository carry a Claude co-author trailer
while the decisions, the measurements, and the judgements about what the numbers mean are
mine ([README.md](../README.md), "How this was made").

## 6. Workshop for PhD and postdoc researchers

*A proposal. Not yet delivered.*

**Using Claude in Research Without Losing Scientific Verification** — 50 minutes.

Learning objectives. By the end, a participant can (1) classify any number in their own
pipeline as evidence or testimony and say who wrote the code that produced it; (2) name the
minimum read-back an experiment must emit — effective config, split hashes, test-evaluation
count, epochs actually run, selection basis; (3) write a test that makes an existing
automated check fail, and explain why a check never observed failing has not been observed
working; (4) identify a comparison whose independent variable is not instantiated in code.

Structure. 10 min — three artifacts on screen, each self-reporting success, one of which is
false; participants vote before the reveal. 15 min — the evidence/testimony line, and the
read-back block. 20 min — hands-on. 5 min — where the boundary sits in each participant's
own work.

Hands-on exercise. Participants receive two `result.json` files from this archive: the
CIFAR-10 anchor and its generated paper. The paper asserts a 3-epoch budget. Working only
from the result file, they must determine how many epochs ran, then write the four-line
read-back block that would have made the answer readable instead of derivable. The intended
outcome is the discovery that `avg_epoch_time_s == total_train_time_s`, which is a fact
about arithmetic and not about trust.

## 7. Limits

The scope limits in [README.md](../README.md) apply here undiminished and are not softened
by this document: not a controlled benchmark; one framework completed; the improvement loop
off; the pre-registered protocol governing work from this repository's first commit onward
and **not** covering the archived runs, which were analysed retrospectively.

Two limits specific to this document:

**Some published counts could not be reproduced, and the headline one has no single
answer.** The repository reports 271 run directories, a 29-run measurement table, 16 of 16
failed experiments producing papers, and 77 result files. Recounting gives this:

| Reported | Recount |
|---|---|
| 96 result files; 122 `run_*` and 91 `v3_*` | reproduced exactly |
| 271 run directories | **definition-dependent.** `outputs/` holds 271 entries, **270** directories (the extra entry is `grep.exe.stackdump`, 1,013 bytes), **224** directories containing any file, and **276** ids with content across `outputs/` and `runs/` |
| the 29-run table | not reproducible — **44** directories carry a non-empty metrics block, and no filter tried yields 29 |
| 16 of 16 papers | not reproducible, and neither is the replacement — **37** directories hold a `paper.md` (reproduced exactly), but how many lacked a successful execution has no single answer. See below |
| 77 result files | not reproducible — 96 `result.json`, 342 `result*.json` |

The counting scripts behind the published figures are in neither repository, so the
differences cannot be adjudicated. Note what the middle row means: "run directories" was
never a count of runs, and three defensible definitions of it disagree — one of them
*higher* than the published figure. This document therefore uses figures it re-derived, and
names the definition whenever it uses one.

**The replacement figure did not survive either.** The 2026-09-09 recount replaced "16 of 16"
with "18 of 37 with no successful execution." A second pass on 2026-09-21 could not produce 18
from the archive under any definition it tried, and no script survives that produces it. What
the archive does support is the table below — every row recomputable from the run directories.

| Definition of "no successful execution" | Of the 37 |
|---|---|
| the run's own `run_summary.json` verdict | **0** — all 37 record `status: completed` with an empty `error` |
| no `result.json` anywhere in the run | **15** |
| no `result*.json` anywhere in the run | **13** |
| no non-empty `metrics` block in any `result.json` | **20** |

```bash
# from legacy_pre_prereg/ ; the metrics predicate below also reproduces the published 44
python - <<'EOF'
import json, os
paper = sorted({dp.split(os.sep)[1] for dp, dn, fn in os.walk('outputs') if 'paper.md' in fn})
def has(d, pat, need_metrics=False):
    for dp, dn, fn in os.walk(os.path.join('outputs', d)):
        for f in fn:
            if not (f == 'result.json' if pat == 'exact' else f.startswith('result') and f.endswith('.json')):
                continue
            if not need_metrics:
                return True
            try:
                m = json.load(open(os.path.join(dp, f), encoding='utf-8'))
            except Exception:
                continue
            if isinstance(m, dict) and m.get('metrics'):
                return True
    return False
print(len(paper), 'runs hold a paper.md')
print(sum(not has(d, 'exact') for d in paper), 'with no result.json')
print(sum(not has(d, 'glob') for d in paper), 'with no result*.json')
print(sum(not has(d, 'exact', True) for d in paper), 'with no non-empty metrics')
EOF
```

18 is not among them, so it is withdrawn rather than restated. Note what the first row costs:
by the pipeline's own verdict **nothing failed at all**, while every evidence-based definition
puts the number between 13 and 20. That gap is the finding of section 3, measured on the
write-up instead of the runs.

That a number in a study record cannot be recomputed makes it testimony — the thesis of
section 4 turned on its author, and the reason this is disclosed rather than reconciled
quietly. The recount is now summarised at the top of [README.md](../README.md) and in
[evidence/INDEX.md](../evidence/INDEX.md), with the original figures left in place there and
in [findings/](../findings/).

**Frequencies are unknown.** Every pathology above is an existence proof from an
uncontrolled archive. None of them supports a rate, and none supports a claim about agent
systems in general.

## 8. Evidence map

| Claim | Evidence path | Type |
|---|---|---|
| Seven framework call sites, each one agent and one task | `MARS/crewai_prototype/phases/phase1_planning.py:212,231`; `phase3_execution.py:1231,1309`; `phase4_writing.py:343,503,575` | verified |
| The largest phase does not import the framework | `MARS/crewai_prototype/phases/phase2_coding.py` (1,209 lines, no `crewai` import) | verified |
| Completed implementation is 14,613 lines of Python | `MARS/crewai_prototype/` | verified |
| Archive holds 270 directories under `outputs/`, 224 of them non-empty, and 96 `result.json` files | `legacy_pre_prereg/outputs/`; [evidence/INDEX.md](../evidence/INDEX.md) | verified |
| A 1,620-fit grid search produced a result bit-identical to the untuned anchor | `legacy_pre_prereg/outputs/*_dce015/…/result.json` vs `*_c0dad6/…/result.json`; grid at `*_dce015/workspace/src/exp_config.py:55-64` | verified, recomputed 2026-09-15 |
| A generated paper asserts 3 epochs; its result file shows 1 | `…run_20260730_160553_*_d359ba/workspace/results/result.json` (`avg_epoch_time_s == total_train_time_s`) and `…/paper/paper.md` | verified |
| The goal gate can pass below the goal on a fraction/percent mismatch | `MARS/crewai_prototype/orchestration/target_gate.py` (`evaluate`) | verified |
| 306 distinct metric key names across 44 metrics-bearing runs; most frequent real name occurs 3× | recount of `legacy_pre_prereg/outputs/**/result.json` | verified |
| 23 of those 44 runs self-report success; 1 names a validation split, and it is from the uncitable dev era | same recount; `legacy_pre_prereg/README.md` | verified |
| Pre-registration makes the harness, not self-report, the arbiter | [protocol/preregistration.yaml](../protocol/preregistration.yaml) (`success_determined_by`) | verified |
| Two defects surfaced only after archiving: one in a seam between individually correct functions, one in a branch no run exercised | `MARS/crewai_prototype/runtime/liveness.py:74-87` vs `:203-204`; `phases/phase3_execution.py:294-315` | verified |
| The framework-comparison claim was withdrawn on structural grounds | [findings/06](../findings/06-why-this-stopped.md); `MARS` commit `eea4b96` | verified |
| Archived runs used OpenAI `gpt-5.2`; no Claude model appears in the archive | model strings in `legacy_pre_prereg/`; `MARS/crewai_prototype/config.yaml` | verified |
| Separating evidence from testimony is the transferable result of the project | [findings/02](../findings/02-evidence-vs-testimony.md) | interpretation |
| Silent success is more dangerous than crashes in autonomous pipelines | [findings/03](../findings/03-silent-success.md), [findings/04](../findings/04-papers-without-results.md) | interpretation |
| The published 29-run table, 16-of-16 and 77-result-file counts are not reproducible; the published 96 and 122 / 91 reproduce exactly; the published 271 is definition-dependent | recount in section 7; no counting script in either repository | verified |
| Division of labour: assistant for critique, navigation, drafting; evidence layer independently verified | section 5; repository commit trailers | proposal |
| The 50-minute workshop and its exercise | section 6 | proposal |
