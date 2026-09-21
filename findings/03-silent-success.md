# 03 — The dangerous failures report success

**Claim.** In an autonomous pipeline, the failures that matter are not crashes. They are
the runs that complete, report success, and are wrong. `exit code 0` is not a success
signal.

**Confidence: high.** Seven distinct instances, each independently confirmed. Five were
found while the system was running; two more surfaced afterwards, while writing this up.

---

## The five

### 1. A search that never ran

A `GridSearchCV` was declared over **324 hyperparameter configurations with 5-fold
cross-validation — 1,620 fits**. The result matched the no-search baseline **to sixteen
decimal places**.

> **Corrected 2026-09-15.** This was published as "810 trials" from the first commit
> until now. The figure came from an internal review note that counted the grid as 162
> combinations; the grid in the run's own `exp_config.py` has 324, because one binary
> axis (`max_features: [None, "sqrt"]`) was dropped in that count. Nobody recomputed it
> before publishing. The unit was wrong too — this is an exhaustive grid search, not a
> trial-based one; there is no `n_trials` anywhere in the run.
>
> ```bash
> # 324 = 3 × 3 × 3 × 2 × 3 × 2, from the run's own generated config
> sed -n '55,64p' <archive>/outputs/*_dce015/workspace/src/exp_config.py
> ```

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

## Two more, found while writing this up

Both were found in August 2026, after the project was archived, while preparing this
repository. Neither had ever fired in production — one because the affected platform was
never used, the other because the affected filesystem was never used. They belong here
because they are the same failure class, and because finding them *after* declaring the
list complete is itself the point.

### 6. A function that does the thing its own docstring says is broken

`crewai_prototype/phases/phase3_execution.py`, `_kill_process_tree`:

```python
def _kill_process_tree(proc) -> None:
    """Kill the child's children too.

    proc.kill() only kills the direct child, so grandchildren such as DataLoader
    workers are left orphaned holding the GPU (observed: after an anchor timeout,
    the experiment process kept running).
    """
    if os.name == "nt":
        subprocess.run(["taskkill", "/F", "/T", "/PID", str(proc.pid)], ...)
    else:
        proc.kill()          # ← exactly what the docstring says does not work
```

The docstring states the problem correctly, cites a real observation, and then the
non-Windows branch does the thing it just ruled out. On Linux and macOS the grandchildren
this function exists to kill survive it.

It never surfaced because every run was on Windows. The correct fix is one code path for
all platforms — enumerate the tree with `psutil`, terminate, wait, then kill — rather than
a per-OS branch where only one branch was ever exercised.

**This is [ADR-013](../decisions/ADR-013-phase3-success-criterion-rc-vs-result-json.md)'s
pattern again**: the conclusion was reached in writing and implemented halfway.

### 7. Two functions, each correct, whose composition is not

`crewai_prototype/runtime/liveness.py`. The lock helper is careful:

```python
except Exception:
    # A filesystem that does not support locking (some network mounts,
    # container volumes). Do not lie and claim we acquired it — return
    # False so this is treated as undetermined.
    return False
```

The caller reads that `False` differently:

```python
if not _try_lock(fh):
    return ALIVE
```

The helper's comment says *undetermined*. The caller says *alive*. On a filesystem without
lock support — a Google Drive mount, some container volumes — every run reads as
permanently ALIVE and can never be reclaimed.

Neither function is wrong on its own. The defect lives in the seam, which is where a
reviewer reading either file in isolation will not find it. And the three-valued design
that makes this finding's fix possible — `alive` / `dead` / **`unknown`** — was already
there; the caller just never used the third value for this case.

## The shape they share

| | |
|---|---|
| Process exit code | 0 |
| Pipeline status | complete |
| Self-reported outcome | success |
| Actual outcome | wrong, and undetectably so from the artifacts |

Four of the first five were found by hand, by noticing something odd in a number. None
were found by the system, and two more were still waiting to be found after the list was
declared complete. That ratio is the finding.

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
| Docstring vs branch | One code path for all platforms, or a test per branch |
| Undetermined read as alive | Three-valued liveness, and callers that handle the third value |

Every one of these is cheap. None of them is clever. They were absent not because they
were hard but because nothing in the development loop ever asked whether a passing check
*could* fail.

## Transferable

For every automated check in an agent pipeline, write a test that makes it fail. A check
that has never been observed failing has not been observed working. And separate two
questions that are easy to conflate: *did the process exit cleanly* and *did the thing
it claimed to do happen*.
