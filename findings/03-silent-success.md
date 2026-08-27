# 03 — The dangerous failures report success

**Claim.** In an autonomous pipeline, the failures that matter are not crashes. They are
the runs that complete, report success, and are wrong. `exit code 0` is not a success
signal.

**Confidence: high.** Five distinct instances, each independently confirmed.

---

## The five

### 1. A search that never ran

A hyperparameter sweep was configured for **810 trials**. The result matched the
no-search baseline **to sixteen decimal places**.

The configuration had no path to reach the experiment — the argument was accepted by the
launcher and never forwarded. The sweep never executed once. The system recorded the run
as *fully verified*.

Detected only because two numbers were suspiciously identical. Nothing in the pipeline
noticed.

### 2. Epoch counts silently downgraded

Runs planned for 3 epochs ran 1. No warning, no event, no note in the result file. The
only way to establish the real number was to count the entries in a per-epoch timing array
— a side effect, not a record.

### 3. Percent against decimal

A target of 70% was compared against a value of `0.6886` after a unit conversion that
happened on one side only. **68.86 was accepted as reaching 70%**, and early-stopping
fired.

A goal check that reports success below the goal is worse than no goal check, because it
consumes the budget that would otherwise have kept running.

### 4. Test data in the validation slot

One run placed the test set into the validation position and reported the resulting number
under the name *validation accuracy*.

Nothing was malformed. The value was real, the label was wrong, and **the label is all a
downstream consumer sees.** This is the case that makes name-based checking hopeless: the
only way to catch it is a content hash of each partition.

### 5. A guard that never fired

A check existed to warn when the executed scale fell below the planned scale. It read the
value from the wrong key. **It fired zero times across the entire archive.**

A dormant guard is worse than a missing one. A missing guard is a known gap; a dormant
guard is a false assurance that something is being watched.

## The shape they share

| | |
|---|---|
| Process exit code | 0 |
| Pipeline status | complete |
| Self-reported outcome | success |
| Actual outcome | wrong, and undetectably so from the artifacts |

Four of the five were found by hand, by noticing something odd in a number. None were
found by the system. That ratio is the finding.

These map to the verification-failure category of the multi-agent failure taxonomy in
[Cemri et al., 2025](https://arxiv.org/abs/2503.13657) — a category which, in a corpus of
1,600+ traces across seven frameworks, is where a large share of failures land.

## Why an autonomous pipeline makes this worse

A human running an experiment notices that a sweep finished too fast. An autonomous
pipeline does not; it reads the result file, sees a number, and writes it up. Each stage
trusts the previous stage's report because a report is all it receives.

So the errors do not stay local. They propagate into the analysis, and from there into the
generated paper — which is [04](04-papers-without-results.md).

## What would have caught each one

| Failure | Check |
|---|---|
| Phantom search | Fixed layer writes back the configuration it actually received |
| Epoch downgrade | Fixed layer counts epochs and records the number |
| Unit confusion | Fixed layer records the unit; comparison normalises before comparing |
| Split contamination | Content hash per partition; overlap is a rule violation |
| Dormant guard | The guard has its own test, with a case that must trip it |

Every one of these is cheap. None of them is clever. They were absent not because they
were hard but because nothing in the development loop ever asked whether a passing check
*could* fail.

## Transferable

For every automated check in an agent pipeline, write a test that makes it fail. A check
that has never been observed failing has not been observed working. And separate two
questions that are easy to conflate: *did the process exit cleanly* and *did the thing
it claimed to do happen*.
