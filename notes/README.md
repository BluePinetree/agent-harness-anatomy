# Notes

Study notes on how existing systems in this space are built. These are reading notes, not
findings — the findings are in [findings/](../findings/).

## claude-code-analysis

[claude-code-analysis/](claude-code-analysis/) — five documents written in April 2026,
analysing how a production coding-agent harness is structured: the entry point and core
loop, the tool system, the service and multi-agent layer, and the UI/state layer, plus four
end-to-end traces following data through the system for different kinds of task.

**About 2,600 lines of prose analysis and hand-drawn structure diagrams.**

### Provenance

These were written while reading a source tree whose own licence file declared it leaked
proprietary code and not redistributable. That tree has been deleted (it was never
committed — verified as zero objects in any ref), and **all 46 verbatim source excerpts have
been removed from these documents.** What remains is my own writing and my own diagrams.

Official documentation for the system analysed:
<https://docs.claude.com/en/docs/claude-code>

### What these notes changed

Three conclusions from this reading fed directly into how the main project was judged:

1. **The harness layer is not a place to build.** Tool loops, permissioning, and context
   management are mature in existing systems. Rebuilding them wins nothing.
2. **A single asynchronous loop is simpler and more robust than multi-agent coordination**
   for linear work — which matches what the measurements later showed
   ([findings/01](../findings/01-single-agent-wrapper.md)).
3. **The verification layer is the gap.** These systems verify code quality, security, and
   task completion. None of them verify whether a claimed experimental result is real.

That third point is why the successor project is narrow
([findings/06](../findings/06-why-this-stopped.md)).
