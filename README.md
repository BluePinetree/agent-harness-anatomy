[English](README.md) · [한국어](README.ko.md)

# agent-harness-anatomy

**What I found by building an autonomous ML-research pipeline on a multi-agent LLM framework, running it 271 times, and then taking it apart.**

![status: inactive](https://img.shields.io/badge/status-inactive-lightgrey)
![license: MIT](https://img.shields.io/badge/license-MIT-blue)

> **Status.** This is a study record, not a maintained project. The system it describes
> (`MARS`) is archived — see [SOURCE.md](SOURCE.md) for the original code and its full
> commit history. What lives here is the analysis: what was actually load-bearing, what
> broke, and why the original research framing turned out to be untestable.

---

## The five things worth knowing

1. **The framework was never doing multi-agent coordination.** Every `Crew(...)` construction
   in the codebase was one agent and one task. The coordination lived in a
   hand-written orchestrator. Grep-verifiable, and it invalidated the study's own
   independent variable. → [findings/01](findings/01-single-agent-wrapper.md)

2. **Separating evidence from testimony is the transferable idea.** Numbers written by
   LLM-authored code are claims. Only code the LLM is forbidden to rewrite produces
   evidence. Every reliability problem below traces back to this line being absent.
   → [findings/02](findings/02-evidence-vs-testimony.md)

3. **The dangerous failures are the ones that report success.** A hyperparameter search
   configured for 810 trials that never executed — and was recorded as "fully verified."
   A goal of 70% marked as reached at 68.86%. Epoch counts silently downgraded from 3 to 1.
   `exit code 0` is not a success signal. → [findings/03](findings/03-silent-success.md)

4. **Every failed experiment still produced a paper.** 16 out of 16. The writing stage had
   no dependency on the execution stage succeeding.
   → [findings/04](findings/04-papers-without-results.md)

5. **The real engineering was keeping 12-hour runs alive**, not the agent logic. Pipe
   deadlock, orphaned GPU processes, session cleanup killing live runs, a timeout
   misdiagnosed as a code bug and retried three times. None of this appears in
   short-horizon agent benchmarks. → [findings/05](findings/05-long-horizon.md)

---

## What the system was

A pipeline that took a research question in natural language and produced a paper:

```
question → plan → design → code → execute → analyze → write
```

Built on CrewAI, with partial ports to AutoGen and LangGraph for a planned three-way
comparison. ~14,500 lines of Python, a React streaming UI, and five human-approval gates.

**It worked.** 271 run directories, 96 result files, papers generated end to end. The
question this repo answers is not *did it run* but *what did running it 271 times show*.

---

## What was actually load-bearing

The interesting finding is a negative one about the framework layer.

| Layer | Lines | What it actually did |
|---|---|---|
| Hand-written orchestrator | 2,350 | **Sequenced the phases. This was the multi-agent system.** |
| Phase logic | 3,755 | Prompt construction, output parsing, repair loops |
| Long-horizon execution | ~1,400 | Process supervision, liveness, crash-resume |
| Scaffold generator | 1,018 | Produced the code the LLM was forbidden to rewrite |
| **The framework** | — | **Wrapped a single LLM call, seven times** |

Measured, not inferred: `Crew(...)` appears at seven call sites, and every one of them is
`Crew(agents=[task.agent], tasks=[task])` — one agent, one task. No sequential process,
no hierarchical delegation, no agent-to-agent messaging. The code-generation phase, the
most complex one, does not import the framework at all; it calls the model directly.

**Consequence for the original research.** The study was designed to compare CrewAI,
AutoGen, and LangGraph with all other variables held constant. But if each framework's
distinctive mechanism is never invoked, all three implementations reduce to *calling the
model in a fixed order* — the same program with different import statements. The
independent variable did not exist in the code. That is why the comparison was withdrawn,
and it is a more interesting result than the comparison would have been.

---

## Evidence vs testimony

The single idea from this project I would carry into any other agent system.

| | Written by | Trust |
|---|---|---|
| **Testimony** | LLM-authored experiment code | None. Record it, never adjudicate on it |
| **Evidence** | Code the LLM may not rewrite | The only admissible basis for a verdict |

**Why the boundary is necessary, measured.** Across 29 archived runs with metrics:

| | |
|---|---|
| Runs recording the configuration actually received | **1 / 29** |
| Runs reporting any validation-set metric | **0 / 29** |
| Runs from which the epoch count can be derived | **3 / 29** |
| Runs that self-reported success | **20 / 29** |

And the result files have no schema at all, because LLM-generated code writes them. Across
29 runs the *most common metric key name appears twice*. Some `metrics` blocks contain
`error`, `traceback`, `host`, `cwd`. **Parsing the agent's own output cannot be made
reliable — the schema is authored fresh on every run.**

The fix is structural, not statistical: a fixed code layer owns the metric namespace and
writes a separate block; anything the LLM names stays in testimony, where it is ignored.

---

## Long-horizon execution

Each mechanism below exists because a multi-hour run died in a way that was not obvious
from the logs.

| Mechanism | The failure that motivated it |
|---|---|
| Separate stdout/stderr reader threads | Pipe buffer filled; the run deadlocked with no error |
| Process-tree termination | Killing the parent left GPU-holding children alive |
| Stall watchdog + heartbeat | 21 hours with no events and no way to tell live from hung |
| File-lock liveness (alive / dead / **unknown**) | PID checks lie; on Windows, signalling a PID can hit your own console |
| Crash-resume checkpoints | Multi-hour work lost to a single interruption |
| Bounded output tails | One event grew to 1.6 GB |

**The sharpest one is not a mechanism.** A run exceeded its 90-minute limit. The analysis
stage diagnosed the cause correctly — *the plan is too large for the time budget* — and
then retried the identical configuration three times, burning 4.5 hours, because nothing
in the harness let it change the budget. That is not a capability failure. The agent knew
the answer and had no lever.

---

## Measured pathologies

These map onto the verification category of the multi-agent failure taxonomy in
[Cemri et al., 2025](https://arxiv.org/abs/2503.13657).

| Pathology | How it presented | How it was caught |
|---|---|---|
| Phantom search | 810-trial sweep configured, never executed | Result identical to the no-search baseline to 16 decimal places |
| Silent scale downgrade | 3 epochs planned, 1 run | Epoch count derived from per-epoch timing arrays |
| Unit confusion | 68.86 accepted against a 70% goal | Early-stop fired when it should not have |
| Split contamination | Test data placed in the validation slot, reported as "validation" | Read the generated code; **the name alone cannot reveal this** |
| Unverifiable test access | Times the test set was evaluated | **Not recorded in any of 77 result files** — unknowable in principle |
| Papers without results | 16 of 16 failed experiments produced a paper | Cross-referencing execution status against artifacts |
| Dormant guard | A scale-check that never fired once | Read the value from the wrong key |
| Docstring vs branch | A tree-kill whose non-Windows path does what its own docstring rules out | Read after archiving; never surfaced because every run was on Windows |
| Undetermined read as alive | A lock helper returning "cannot tell" that the caller reads as "alive" | Read after archiving; needs a filesystem without lock support |

**The last row is the theme of this repo.** The failure mode is not a missing check. It is
a check that exists, is reported as passing, and is structurally incapable of firing.

---

## Why it stopped

The original contribution was a three-framework comparison under a pre-registered
protocol. Three things ended it, in order of severity:

1. **The independent variable was never instantiated** (above). No amount of additional
   running would have produced a difference to measure.
2. **The design could not discriminate even in principle.** Three runs per task yields
   Wilson intervals of [0.00, 0.56] for 0/3 and [0.44, 1.00] for 3/3 — overlapping. And
   the pass threshold allowed 2.00 pp of tolerance against 16–23 pp of measured
   run-to-run variance, which pre-determines the primary metric to zero.
3. **Each stated contribution was already published, at larger scale**, by others.

What survived is the instrumentation problem: agent systems report success they cannot
substantiate, and their own artifacts are insufficient to check. That is now a separate
project. → [findings/06](findings/06-why-this-stopped.md)

---

## Scope and limits

Stated up front rather than discovered by a reader:

- **n = 271 run directories, 96 result files.** One operator, one machine, one model
  family, three task families (image classification, tabular regression, time series).
- **Not a controlled benchmark.** Conditions changed across the period as the system was
  debugged. The pathologies are existence proofs, not rates.
- **One framework completed.** AutoGen and LangGraph were partially ported and never ran
  end to end.
- **The improvement loop was off** for all three tasks. The system does not search over
  methodology; it implements a specified one.
- **Pre-registration timing.** The protocol in [protocol/](protocol/) governs work from
  this repository's first commit onward. It does **not** cover the archived runs, which
  were analysed retrospectively.

### How this was made

The decisions here are mine — what to build, what to measure, what the numbers meant, and
when to stop — and I shaped the structure of both the system and this write-up. Repetitive
code and skeleton scaffolding were delegated to AI, along with much of the drafting. Commits
carry a co-author trailer accordingly.

---

## Repository map

| Path | Contents |
|---|---|
| [findings/](findings/) | One claim per document, each with an evidence pointer |
| [notes/](notes/) | Study notes on how existing coding-agent harnesses are structured |
| [decisions/](decisions/) | 14 architecture decision records from the original project |
| [log/](log/) | Index of the dated Korean working documents (primary sources) |
| [evidence/](evidence/) | Inventory of the archived runs |
| [protocol/](protocol/) | The pre-registered measurement protocol |
| [SOURCE.md](SOURCE.md) | The archived original repository — code and full commit history |

## Reading paths

- **Architecture lesson:** findings/01 → findings/02 → notes/
- **Reliability engineering:** findings/03 → findings/05 → decisions/
- **Research-methods lesson:** findings/06 → protocol/

## Related work

- [Cemri et al., *Why Do Multi-Agent LLM Systems Fail?* (2025)](https://arxiv.org/abs/2503.13657) — the failure taxonomy these pathologies map onto
- [CORE-Bench](https://arxiv.org/abs/2409.11363) — computational reproducibility as an agent task
- CrewAI, AutoGen, LangGraph — the frameworks under study

## License

MIT for code, and the documents are mine to share. See [LICENSE](LICENSE).
