# 01 — The multi-agent framework was wrapping a single agent

**Claim.** In this system, the multi-agent framework never performed multi-agent
coordination. Every framework invocation ran one agent with one task. The coordination was
done by a hand-written orchestrator.

**Confidence: high.** Verifiable with `grep` in the archived repository.

---

## The evidence

Seven call sites construct the framework's execution unit. All seven take the same shape:

```python
Crew(agents=[task.agent], tasks=[task], verbose=False).kickoff()
```

Verify it yourself:

```bash
grep -rn "agents=\[" crewai_prototype/phases/
```

(A single-line pattern such as `Crew(agents=\[` returns six — one call in
`phase3_execution.py` wraps the constructor across two lines and escapes it.)

One agent. One task. No sequential process, no hierarchical delegation, no
agent-to-agent messaging — none of the mechanisms that distinguish the framework from a
plain model call.

Tool usage is similarly lopsided:

| Phase | Tools passed |
|---|---|
| Planning | `tools=[]` — **both agents, none at all** |
| Execution | Command runner, result reader, file editor, syntax checker — **the only real use** |
| Writing | One file reader, one report writer |

And the largest phase — code generation, ~1,200 lines — **does not import the framework**.
It calls the model directly.

## What the framework was actually providing

| | Provided | Substantial? |
|---|---|---|
| Role/goal text assembled into a system prompt | yes | no — string formatting |
| A tool-calling loop | yes | **yes — the only real dependency** |
| A model-call wrapper | yes | no — a thin adapter |

One of three. Everything else attributed to the framework was in the project's own code.

## Where the coordination actually lived

A single orchestrator module called each phase in sequence:

```
run_planning_phase → run_coding_phase → run_execution_phase → run_writing_phase
```

Roughly 2,350 lines. It owned the repair loop, the budget escalation, the event stream,
the cancellation token, and the structured handoffs between phases.

**So the system was multi-agent** — six role-specialised agents with distinct prompts,
distinct tools, and structured handoffs. That is a multi-agent system by any ordinary
definition. It just was not the *framework's* multi-agent system.

## Why this was easy to miss for months

Nothing failed. The pipeline ran end to end, produced papers, and the framework appeared
in every phase's import block. The framework was present, invoked, and load-bearing for
one thing — the tool loop — which is enough to make it feel foundational.

The way it surfaced was mundane: counting the call sites while estimating the cost of
replacing the framework. The estimate came back *cheap*, and the reason it was cheap was
that almost nothing depended on it.

## What follows

Two things, one practical and one fatal.

**Practical:** removing the framework was a few days of work, not a rewrite. What replaces
it is a model adapter plus roughly 200 lines of tool loop. Fewer lines than before.

**Fatal:** the study this system was built for compared three frameworks with all other
variables held constant. If each framework's distinctive mechanism is never invoked, all
three implementations reduce to *calling the model in a fixed order* — the same program with
different import statements. See [06](06-why-this-stopped.md).

## Transferable

Before attributing a property of your system to a framework, count the call sites and read
what they pass. A dependency that appears in every file can still be doing one small thing.
The question is not whether the framework is imported but which of its mechanisms are
reachable from your code path.
