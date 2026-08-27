# Working documents (primary sources)

The dated record of how the project was actually built and reasoned about — design specs,
review sessions, campaign logs, decision notes. **These are unedited.** Their value is that
they were written before the outcome was known, so nothing here has been corrected in
hindsight.

Mostly Korean. Translating them would mean rewriting them, and rewriting a primary source
destroys the only property that makes it worth keeping. This index carries the English gloss
instead.


> **Links inside these documents are broken, deliberately.** They were written against the
> original repository's layout and point at paths like `docs/decisions/ADR-001…`,
> `crewai_prototype/`, and `examples/` — none of which exist here. Rewriting them would mean
> editing primary sources, which is exactly what makes a record like this worth keeping.
> To follow one, use the archived original: [SOURCE.md](../SOURCE.md).

---

## `specs/` — design specifications (2026-05, English)

Written in a single burst on 2026-05-19 before the V4 implementation, plus later additions.

| File | What it is |
|---|---|
| `INDEX.md` | Entry point for the spec set |
| `DESIGN_MEETING.md` | Design-meeting conclusions — the circuit-breaker → escalation decision that shaped Phase 2 |
| `ARCHITECTURE.md` | Authoritative architecture spec |
| `PIPELINE_SPEC.md` | Five-phase state machine (983 lines) |
| `MODULE_SPEC.md` | Module and signature spec (1,371 lines) |
| `AGENT_SPEC.md` | Per-agent role, schema, and prompt spec |
| `ERROR_RECOVERY_SPEC.md` | Error taxonomy and escalation spec |
| `CONSTANTS.md` | Constants and tuning guide |
| `API_SPEC.md` | REST/SSE API reference (2026-06-01) |
| `test_checklist_v1.md` | L0–L4 manual test checklist. *Local paths replaced with `<HOME>`* |

## `working/` — dated working records (2026-05 → 2026-08)

| File | What it is | Date |
|---|---|---|
| `expert_review_crewai_stability_ko.md` | Stability design review in dialogue form (1,497 lines) | 05-26 |
| `implementation_plan_crewai_stability_ko.md` | Implementation plan from that review | 05-27 |
| `structural_review_ko.md` | Complexity audit of the plan above | 05-28 |
| `frontend_implementation_plan_ko.md` | UI implementation plan | 05-28 |
| `patch_notes_2026_06.md` | June bug-fix notes with literature references | 06 |
| `insights/success_signal_reliability.md` → `success_signal_reliability.md` | Literature note on the success-signal reliability gap — **the origin of findings 02 and 03** | 06-23 |
| `PUBLIC_REPO_PLAN.md` | Public-repo plan. **Self-marked obsolete 2026-08-14** — kept to show what was believed in June | 06-04 |
| `stage1-3_crewai_anchor_plan_ko.md` | Stage 1–3 merge and anchor-run execution plan | 07-21 |
| `s2_expert_panel_review_ko.md` | Four-lens review of the CIFAR-100 run | 07-24 |
| `campaign_A_pipeline_fixes_ko.md` | Pipeline and tool fix campaign tracker | 07-24 |
| `p1-3_env_reproducibility_ko.md` | Environment selection and reproducibility notes | 07-27 |
| `ui_option_selection_campaign_ko.md` | UI "choices" pattern campaign | 07-27 |
| `env_and_provider_changes_ko.md` | LiteLLM → native provider change log | 07-27 |
| `data_ingestion_tooluse_design_ko.md` | Data-ingestion design session | 07-28 |
| `research_quality_review_ko.md` | Self-critical quality review, with 08-04 corrections appended | 07-30 |
| `spectrum_test_log_ko.md` | S1–S10 spectrum run log | 08-04 |

## `root/` — documents from the original repository root

| File | What it is |
|---|---|
| `PROJECT_NOTE.md` | **The earliest note (2026-04-14)** — where the three-framework idea came from |
| `COMPARISON_PLAN.md` | The withdrawn study's full plan (870 lines), 2026-04-20 |
| `PHASE1_DESIGN.md` | CrewAI-native redesign plan |
| `IMPLEMENTATION_PLAN.md` | Plan borrowing a coding-agent architecture. *Source citations sanitized — see the header* |
| `crewai_claude_structure_diff_ko.md` | Structure verification, three-role format (465 lines) |
| `crewai_vs_claude_structure_analysis_ko.md` | Shorter report on the same subject; superseded by the above |
| `DEV_NOTES.md` | Cross-framework pitfalls notebook — the strongest technical note at root |
| `README.md` | The public README as of June 2026. **States the withdrawn claim as fact** — that is why it is here |
| `CONTRIBUTING.md` | Contributor guide. Frames the project as active |
| `CHANGELOG.md` | Version history through v0.4.0 |
| `COMMIT_CONVENTION.md` | Commit message convention |
| `CLAUDE.md` / `AGENTS.md` | Agent instruction files — the record of how the harness itself was driven. Near-identical to each other |

---

## What is not here

Some working documents were held back. They contain venue and submission material for work
that was never submitted, machine identifiers, or pointed criticism of named third parties.
None of that serves a reader, and publishing it can only cost.

Held back: the DAI/AAMAS submission plans and publication roadmap, the 2026-08 planning
panel record, the seminar deck design and its revision plan, the pre-registration prose
companion, the 2026-08 change log, and the competitive-landscape tracking log.

The binding pre-registration itself **is** published, at [`../protocol/`](../protocol/).

## Reading order

If you want the arc rather than the reference:

```
root/PROJECT_NOTE.md          what this was going to be        2026-04
root/COMPARISON_PLAN.md       the study design                 2026-04
specs/DESIGN_MEETING.md       the decision that shaped Phase 2 2026-05
working/expert_review_...     what broke and why               2026-05
working/success_signal_...    the observation that mattered    2026-06
working/research_quality_...  turning the lens inward          2026-07
../findings/06                where it ended                   2026-08
```
