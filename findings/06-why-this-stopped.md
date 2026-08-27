# 06 — The comparison was untestable by construction

**Claim.** The original research question could not have been answered by this design, and
the reason is structural rather than a matter of unfinished work. Stopping and recording
that is the result.

**Confidence: high.** The primary reason follows directly from
[01](01-single-agent-wrapper.md).

---

## What the study was

Implement one ML-research pipeline three times — on CrewAI, AutoGen, and LangGraph — hold
every other variable constant, and measure whether the framework's design affects
reliability and output quality. Thresholds and selection rules fixed before seeing results.

The reasoning was sound. The execution had three problems, in order of severity.

## 1. The independent variable did not exist in the code

Every framework invocation in the completed implementation was one agent and one task
([01](01-single-agent-wrapper.md)). None of the framework's distinguishing mechanisms were
reachable from the code path.

So all three implementations would have reduced to *calling the model in a fixed order* —
**the same program with different import statements.**

An adversarial review of the design said the conditions had been held so constant that the
agent had no degrees of freedom left. That was generous. The problem was worse: the thing
being compared was never instantiated. No amount of additional running would have produced
a difference, because there was nothing structurally different to produce one.

## 2. The design could not discriminate even in principle

Three runs per task, with *passed all three* as the primary metric. Three runs admits four
outcomes. The Wilson intervals:

| Observed | 95% interval on the true rate |
|---|---|
| 0 / 3 | **0% – 56%** |
| 3 / 3 | **44% – 100%** |

Overlapping. A framework that never succeeds and a framework that succeeds most of the time
are not distinguishable at this sample size. Completing all three implementations — 27 runs
total — yields nine coin flips per framework.

And the pass threshold allowed **2.00 pp** of tolerance, on the grounds that good
human implementations vary by about that much. But **measured run-to-run variance of the
agent was 16–23 pp**, ranging to 43.7 pp. The tolerance was an eighth of the noise. The
primary metric was pre-determined to zero before the first run.

## 3. Each stated contribution had already been published, larger

| Claim | Prior work |
|---|---|
| Pre-registration protocol for agent experiments | Published three months earlier, and a later study applied it across 24 tasks and 4,644 runs |
| Self-reported vs verified success gap (measured here at 21 → 11) | Studied at 11,755 runs across 12 model families, with a detector built |
| Failure taxonomy | 1,600+ traces, 7 frameworks, two independent annotators |
| Controlled framework comparison | Published months earlier, with dramatic effect sizes |

Checked by opening the papers, not by reading abstracts.

## What this is not

It is not "the project failed." The system worked: 271 runs, papers generated end to end,
and the reliability engineering in [05](05-long-horizon.md) is real and was necessary.

What ended was a specific *claim structure*. The pre-registration was honoured — the
protocol said what would count as an answer, and the instrumentation showed the design
could not produce one. That is what pre-registration is for. Discovering it before
publishing is the mechanism working, not failing.

## What survived

One thing, and it is the thing this project accidentally had all along:

> Agent systems report success they cannot substantiate, and their own artifacts are
> structurally insufficient to check.

Findings [02](02-evidence-vs-testimony.md), [03](03-silent-success.md), and
[04](04-papers-without-results.md) are all instances. Zero of 29 runs recorded a validation
metric while 20 claimed success; the number of test-set evaluations was recorded in none of
77 result files and is therefore unknowable in principle.

Checking whether that gap was already filled: the largest open agent-harness project in
this space ships evaluation and verification skills, and its own documentation states that
it records pass/fail outcomes but has **no audit trail proving which parameters were
actually active**, and no notion of data-partition integrity. It verifies software quality.
It does not verify whether a claimed experimental result is real.

That gap is now a separate project. Not a bigger version of this one — a narrower one.

## Transferable

Before committing to a comparison, verify that the independent variable is instantiated in
the code, and compute the discriminating power of your sample size against your own
measured variance. Both checks are cheap and both would have ended this design in an
afternoon. And when a pre-registered design turns out to be unable to answer its question,
publishing that is worth more than quietly changing the question.
