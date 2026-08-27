# 05 — Keeping multi-hour runs alive was the bulk of the engineering

**Claim.** The hard part was not agent logic. It was process supervision for runs lasting
hours. These failure modes do not appear in short-horizon agent benchmarks, and the
literature on long-horizon agents mostly studies a different thing.

**Confidence: medium.** Single operator, one machine. The individual failures are
existence proofs; their frequency is not established.

---

## The mechanisms, and why each exists

| Mechanism | The failure that motivated it |
|---|---|
| Separate stdout/stderr reader threads | Pipe buffer filled and the child blocked writing. The run hung with **no error and no output** |
| Process-tree termination | Killing the parent left children alive, **still holding the GPU** |
| Stall watchdog + periodic heartbeat | **21 hours with no events.** No way to distinguish a long run from a dead one |
| File-lock liveness, three-valued | PID checks lie after reuse. On Windows, signalling a PID can hit your own console. The third value is **`unknown`** |
| Crash-resume checkpoints | Hours of work lost to one interruption |
| Bounded output tails | One event record grew to **1.6 GB** |
| Failure-kind classification | A timeout was diagnosed as a code bug and repaired-and-retried (below) |

None of these are about the model. All of them are about a subprocess that runs for a long
time on a machine that is not idle.

## The one that is not a mechanism

A run exceeded its 90-minute limit. The analysis stage diagnosed the cause **correctly** —
the plan was too large for the time budget — and then retried the identical configuration
three times, burning **4.5 hours**.

Nothing in the harness let it change the budget.

That is not a capability failure. The agent produced the right diagnosis and had no lever
to act on it. The gap was in what the harness exposed, not in what the model knew.

The remedy was to classify the failure kind before entering the repair loop, so that
`timeout` and `stall` short-circuit repair instead of feeding a code-fixing loop a problem
that is not about code. Cheap, and it only became visible after watching the same 90
minutes burn three times.

## Why the third liveness value matters

Most liveness checks are binary: alive or dead. Both wrong answers are expensive here —
declaring a live 12-hour run dead throws away the run; declaring a dead one alive blocks
the slot forever.

So the check returns `alive` / `dead` / **`unknown`**, and `unknown` triggers observation
rather than action. This is also what made the original session-cleanup routine dangerous:
it made a binary decision and could **kill a live multi-hour run** while reclaiming what it
believed were stale sessions.

## How this differs from the published long-horizon literature

Long-horizon agents are an active area. What that work mostly studies is **the model
degrading**: losing track of earlier decisions, declaring half-finished work done,
drifting from the goal.

What is described here is different:

| Published long-horizon work | This |
|---|---|
| The **model** degrades over a long horizon | The **harness** silently corrupts or destroys the run |
| Failure is in reasoning | Failure is in process supervision, budget control, observability |
| Visible in the trace | **Invisible in the trace, and invisible to the evaluation** |

Both are real. The second is under-described, and it is only observable if you actually run
things for hours. A benchmark task that finishes in minutes does not fill a pipe buffer,
does not orphan a GPU process, and does not produce 21 hours of silence.

## Honest limits

- One operator, one machine, one OS. Several of these are Windows-specific in their
  details, though the classes are not.
- Frequencies are unknown. The archive is not a controlled sample; conditions changed as
  the system was debugged.
- Everything here is a fix, not a measurement. There is no before/after quantification of
  reliability, because there was no stable baseline to measure against.

## Transferable

If your agent runs subprocesses for hours, budget engineering time for supervision, not
just orchestration: read both output streams concurrently, kill trees rather than
processes, emit a heartbeat, allow `unknown` as a liveness answer, and classify *why* a run
ended before deciding to retry it. And be suspicious of a reliability story validated only
on tasks that finish in minutes.
