# Architecture decision records

Fourteen ADRs from the original project, carried over unedited. Each records a decision, the
context, and the alternatives considered at the time.

**They are not corrected in hindsight.** Where a later finding contradicts one, the finding
says so and links here; the ADR itself stays as written. A decision record edited after the
outcome is known is not a record.

| ADR | Subject | Later status |
|---|---|---|
| [001](ADR-001-direct-llm-calls.md) | Direct LLM calls instead of framework agents | **Vindicated, and further than intended** — [findings/01](../findings/01-single-agent-wrapper.md) shows this was effectively true system-wide, not just in the coding phase |
| [002](ADR-002-json-handoff.md) | Structured JSON handoffs between phases | Held up |
| [003](ADR-003-sse-streaming.md) | Server-sent events for streaming | Held up |
| [004](ADR-004-staged-code-generation.md) | Staged code generation | Held up |
| [005](ADR-005-hitl-gate-architecture.md) | Five human-approval gates | Held up in design; in practice the harness auto-approved plans, so archived runs record approvals that were machine-generated |
| [006](ADR-006-event-taxonomy.md) | Event taxonomy | Held up |
| [007](ADR-007-event-normalization.md) | Event normalisation | Held up |
| [008](ADR-008-repair-loop-escalation.md) | Repair-loop escalation | **Superseded in part** — the loop bypassed the code gates from the previous phase, and timeouts entered it as if they were code defects ([findings/05](../findings/05-long-horizon.md)) |
| [009](ADR-009-generated-code-import-rules.md) | Import rules for generated code | Held up |
| [010](ADR-010-importlib-sys-modules-registration.md) | Module registration | Held up |
| [011](ADR-011-phase3-analyzer-stderr-context.md) | Give the analyser stderr context | Held up, and necessary |
| [012](ADR-012-phase1-reject-vs-modify-branching.md) | Reject vs modify branching | Held up |
| [013](ADR-013-phase3-success-criterion-rc-vs-result-json.md) | Success criterion: return code vs result file | **The most important one.** Directly anticipates [findings/03](../findings/03-silent-success.md) — and the fact that the pathologies still occurred afterwards shows a written decision is not an enforced one |
| [014](ADR-014-windows-asyncio-proactor-event-loop-blocking.md) | Windows event-loop blocking | Held up |

## The pattern worth noticing

ADR-001 and ADR-013 both reached, in writing and early, conclusions that the measurements
later confirmed. Both were then only partially acted on.

The gap between a recorded decision and an enforced one is where most of
[findings/03](../findings/03-silent-success.md) lives. Writing the decision down is
necessary and is not sufficient; without a test that fails when the decision is violated,
an ADR is a statement of intent.
