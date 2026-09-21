# Application snippets

Reusable text for applications, with an evidence path under each claim. Companion to
[CASE_STUDY.md](CASE_STUDY.md).

**Two rules these snippets follow.** Numbers are the ones re-derived from the archive.
Some of the repository's published counts reproduced exactly and some did not, and the
headline directory count depends on a definition that was never stated (see
[CASE_STUDY.md](CASE_STUDY.md) §7). And the archived runs were driven by OpenAI `gpt-5.2` —
no Claude model appears anywhere in the archive — so every sentence about Claude is in the
present or future tense, about how this experience shapes current practice.

---

## 50-word project summary

An autonomous ML-research pipeline — plan, code, execute, analyse, write — built on a
multi-agent LLM framework, run until its archive held hundreds of run directories and 96
result files, then taken apart. The framework-comparison claim it was built to test was
withdrawn on structural grounds. The verification problem it exposed survived.

> *Evidence:* `legacy_pre_prereg/outputs/` (270 directories, 224 of them non-empty, 96
> `result.json`); [findings/06](../findings/06-why-this-stopped.md); `MARS` commit
> `eea4b96`.

---

## 100-word project summary

I built an autonomous research pipeline that turned a natural-language question into a
paper: plan, design, code, execute, analyse, write. 14,613 lines of Python on CrewAI, a
streaming UI, five approval gates. Counting the framework's call sites ended the study it
was built for: all seven construct one agent with one task, so the three-framework
comparison had no independent variable in the code, and I withdrew it. What remained was
sharper. Generated result files have no schema — 306 distinct metric key names across 44
runs — and papers were written from executions that never succeeded. Verification, not
capability, was the bottleneck.

> *Evidence:* `MARS/crewai_prototype/` (14,613 lines);
> `phases/phase1_planning.py:212,231`, `phase3_execution.py:1231,1309`,
> `phase4_writing.py:343,503,575` (seven `Crew(...)` sites, each one agent and one task);
> [findings/06](../findings/06-why-this-stopped.md); recount of
> `legacy_pre_prereg/outputs/**/result.json` (306 key names, 44 metrics-bearing runs);
> 37 run directories hold a `paper.md`. How many lacked a successful execution has no single
> answer — **0** by the runs' own status fields, **15 / 13 / 20** by three evidence-based
> definitions ([CASE_STUDY.md](CASE_STUDY.md) §7, with the script).

---

## 150 words — How this shaped my use of AI in research

Four months of running an autonomous pipeline taught me one distinction, and I now organise
my work around it: a number written by model-authored code is the model's testimony about a
measurement, not the measurement. One archived run declared a 1,620-fit hyperparameter grid search
and returned a result identical to the untuned baseline in all sixteen digits. Another
produced a paper asserting three training epochs when the arithmetic in its own result file
— average epoch time equal to total training time — is consistent only with one.

So I now use Claude where being wrong is cheap and recoverable: critiquing a design before
it runs, navigating unfamiliar code, reading the seams between functions that are each
correct alone, drafting and translating, and analysing failures in logs I keep. Metric
truth, dataset splits, trial and epoch counts, stopping criteria and statistical verdicts
stay with a fixed layer no model may author, and I verify those independently.

> *Evidence:* `legacy_pre_prereg/outputs/*_dce015/…/result.json` vs `*_c0dad6/…/result.json`
> (both `0.7756446042829697`); `…run_20260730_160553_*_d359ba/workspace/results/result.json`
> (`avg_epoch_time_s == total_train_time_s`) against `…/paper/paper.md`;
> [findings/02](../findings/02-evidence-vs-testimony.md);
> [protocol/preregistration.yaml](../protocol/preregistration.yaml)
> (`success_determined_by: harness`).

---

## Technical credibility

- **Built and operated a six-stage autonomous research pipeline end to end** — 14,613 lines
  of Python, a React streaming UI, five approval gates, and an archive of 270 directories
  under `outputs/` (224 non-empty) and 96 result files. The gates held up in design; in the
  archived runs the headless harness auto-approved plans, so those approvals are
  machine-generated.
  *Evidence:* `MARS/crewai_prototype/`; `legacy_pre_prereg/outputs/`;
  [decisions/README.md](../decisions/README.md) (ADR-005 later status).

- **Diagnosed a framework dependency by measurement rather than assumption.** All seven
  framework call sites construct one agent with one task; the largest phase, 1,209 lines,
  does not import the framework at all. Removing it was days of work, not a rewrite.
  *Evidence:* `MARS/crewai_prototype/phases/phase{1,3,4}_*.py`; `phase2_coding.py`;
  [findings/01](../findings/01-single-agent-wrapper.md).

- **Made multi-hour runs survivable, which was the bulk of the engineering.** Concurrent
  stdout/stderr readers for a pipe deadlock that hung with no error, process-tree
  termination for GPU-holding orphans, a stall watchdog after 21 hours of silence, and
  three-valued file-lock liveness (`alive` / `dead` / `unknown`) after a cleanup routine
  killed live runs.
  *Evidence:* `MARS/crewai_prototype/runtime/liveness.py` (module docstring records the
  incident); [findings/05](../findings/05-long-horizon.md).

## Research integrity and verification

- **Withdrew my own central claim on structural grounds, and published why.** The
  three-framework comparison was untestable by construction: the independent variable was
  never instantiated in the code, and at three rollouts per task the Wilson intervals for
  0/3 and 3/3 overlap. Recording that is the pre-registration working, not failing.
  *Evidence:* [findings/06](../findings/06-why-this-stopped.md); `MARS` commit `eea4b96`
  ("archive MARS and withdraw the framework-comparison claim").

- **Established that this pipeline's success signal was unverifiable, then specified the
  fix.** Of 44 runs with a non-empty metrics block, 23 self-report success while exactly one
  names a validation split — and that one belongs to a development era the archive marks
  uncitable. The remedy is structural: a fixed layer owns the metric namespace and writes
  effective config, per-split content hashes, test-access count and epochs actually run.
  *Evidence:* recount of `legacy_pre_prereg/outputs/**/result.json`;
  [findings/02](../findings/02-evidence-vs-testimony.md);
  [protocol/preregistration.yaml](../protocol/preregistration.yaml).

- **Applied the same scrutiny to my own write-up.** Preparing this material I could not
  recompute three of the repository's published counts — the 29-run table became 44, 16 of
  16 papers became 37 with no single answer for how many had failed, and the 77 result files
  became 96 — while its 96-result-file
  and 122 / 91 generation figures reproduced exactly. The headline "271 run directories"
  turned out to have no single answer: 271 entries, 270 directories, 224 non-empty, 276
  across `outputs/` and `runs/`. The original counting scripts exist in neither repository,
  so the discrepancy is disclosed rather than reconciled — a figure that cannot be
  recomputed is testimony by my own definition.
  *Evidence:* [CASE_STUDY.md](CASE_STUDY.md) §7 and its evidence map.

## Workshop topics for PhD and postdoc researchers

- **Using Claude in Research Without Losing Scientific Verification** (50 min). Where the
  evidence/testimony line falls in a real pipeline, and the minimum read-back an experiment
  must emit. Hands-on: determine the true epoch count of an archived run from its result
  file alone, against a generated paper that asserts a different number.
  *Evidence:* [CASE_STUDY.md](CASE_STUDY.md) §6. **Proposed, not yet delivered.**

- **Writing a Check That Can Actually Fail** (45 min). Every automated check gets a test
  that trips it. Drawn from a scale guard that read the wrong key and fired zero times
  across the archive, and from two defects that survived because one platform and one
  filesystem were never used.
  *Evidence:* [findings/03](../findings/03-silent-success.md). **Proposed.**

- **Is Your Comparison Instantiated?** (45 min). Two cheap pre-flight checks before
  committing to a comparative study: confirm the independent variable is reachable in the
  code path, and compute the discriminating power of your sample size against your own
  measured run-to-run variance rather than an assumed tolerance.
  *Evidence:* [findings/06](../findings/06-why-this-stopped.md);
  [protocol/README.md](../protocol/README.md) ("Known defects in this version").
  **Proposed.**
