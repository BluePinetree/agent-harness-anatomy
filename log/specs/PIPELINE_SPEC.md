# PIPELINE_SPEC.md
# Research Automation Pipeline — Complete State Machine Specification

> Version: 1.0  
> Last updated: 2026-05-19  
> Status: Authoritative specification for implementation

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Top-Level State Machine](#2-top-level-state-machine)
3. [Phase 0: Workspace Setup](#3-phase-0-workspace-setup)
4. [Phase 1: Planning & Design with Approval Gate](#4-phase-1-planning--design-with-approval-gate)
5. [Phase 2: Staged Coding](#5-phase-2-staged-coding)
6. [Phase 3: Experiment Execution](#6-phase-3-experiment-execution)
7. [Phase 4: Section-by-Section Paper Writing](#7-phase-4-section-by-section-paper-writing)
8. [Cross-Phase State Persistence](#8-cross-phase-state-persistence)
9. [User Interaction Protocol](#9-user-interaction-protocol)
10. [Phase Transition Criteria](#10-phase-transition-criteria)

---

## 1. System Overview

The pipeline is a sequential state machine with five phases. Each phase produces typed outputs consumed by the next. User interaction points are explicit and blocking — the system checkpoints state to disk before pausing, so a restart after user input does not re-execute completed work.

```
INPUT: research_topic (str), optional workspace_path (str)
         |
         v
  +------+------+
  |  PHASE 0    |  Workspace Setup
  |  WorkspaceSetupAgent  |
  +------+------+
         |  WorkspaceConfig
         v
  +------+------+
  |  PHASE 1    |  Planning & Design
  |  Planner → Designer → [APPROVAL GATE]  |
  +------+------+
         |  Approved DesignerResult
         v
  +------+------+
  |  PHASE 2    |  Staged Coding (3 stages)
  |  Coder × stages 1/2/3  |
  +------+------+
         |  ExecutionReadyBundle
         v
  +------+------+
  |  PHASE 3    |  Experiment Execution
  |  ExecutorAgent  |
  +------+------+
         |  ExecutorResult
         v
  +------+------+
  |  PHASE 4    |  Paper Writing (7 sections)
  |  WriterAgent × sections  |
  +------+------+
         |
         v
OUTPUT: paper.md / paper.tex, result.json, all artifacts
```

---

## 2. Top-Level State Machine

```
State: PipelineState
  phase:        Literal["setup","planning","design","approval","coding","execution","writing","done","error"]
  checkpoint:   str        # path to checkpoint JSON on disk
  run_id:       str        # UUID4, set at Phase 0
  error_count:  int        # cumulative cross-phase error count
  paused_for_user: bool    # True when blocking on user input

Transitions:
  setup       --setup_ok--->       planning
  setup       --setup_fail--->     error  (unrecoverable: no workspace)

  planning    --plan_ok--->        design
  planning    --plan_fail--->      planning  (retry with error context, max PLAN_MAX_RETRIES)
  planning    --plan_exhausted-->  error

  design      --design_ok--->      approval
  design      --design_fail--->    design  (retry, max DESIGN_MAX_RETRIES)
  design      --design_exhausted-> error

  approval    --approved--->       coding
  approval    --rejected--->       planning  (user feedback injected)
  approval    --modified--->       planning  (user edits injected)

  coding      --coding_ok--->      execution
  coding      --escalated--->      coding  (user provides fix, re-enter coding)
  coding      --skipped--->        coding  (file stubbed, continue)

  execution   --exec_ok--->        writing
  execution   --exec_fail--->      execution  (escalation, not circuit breaker)
  execution   --escalated--->      execution

  writing     --writing_ok--->     done
  writing     --partial--->        done  (some sections NEEDS_REVIEW, flagged in output)
```

---

## 3. Phase 0: Workspace Setup

### State Machine

```
[START]
   |
   v
+--+---------------------------+
|  Validate or create          |
|  workspace directory         |
+--+---------------------------+
   |
   +--[path provided]--> validate_path()
   |                         |
   |                    [exists & writable] --> use_as_root
   |                         |
   |                    [not writable] --> raise SetupError (UNRECOVERABLE)
   |
   +--[no path]----------> generate_default_path()
                               |
                          outputs/{timestamp}_{topic_slug}/
                               |
                          create_directories()
                               |
                          validate_write_permissions()
                               |
                          [fail] --> raise SetupError (UNRECOVERABLE)
                               |
                          [ok]  --> emit WorkspaceConfig
```

### Python Implementation

```python
import uuid, time, re, os
from pathlib import Path
from dataclasses import dataclass

@dataclass
class WorkspaceConfig:
    root_dir: Path
    run_id: str
    session_id: str
    created_at: float
    checkpoint_path: Path

WORKSPACE_SUBDIRS = [
    "src", "results", "logs", "plots", "checkpoints", "paper"
]

def setup_workspace(topic: str, path: str | None = None) -> WorkspaceConfig:
    run_id = str(uuid.uuid4())[:8]
    session_id = str(uuid.uuid4())

    if path:
        root = Path(path).expanduser().resolve()
    else:
        ts = time.strftime("%Y%m%d_%H%M%S")
        slug = re.sub(r"[^a-z0-9]+", "_", topic.lower())[:40]
        root = Path("outputs") / f"{ts}_{slug}"

    root.mkdir(parents=True, exist_ok=True)

    # validate write permissions
    probe = root / ".write_probe"
    try:
        probe.write_text("ok")
        probe.unlink()
    except OSError as e:
        raise SetupError(f"Workspace not writable: {root}: {e}") from e

    for sub in WORKSPACE_SUBDIRS:
        (root / sub).mkdir(exist_ok=True)

    checkpoint_path = root / "checkpoints" / "pipeline_state.json"

    config = WorkspaceConfig(
        root_dir=root,
        run_id=run_id,
        session_id=session_id,
        created_at=time.time(),
        checkpoint_path=checkpoint_path,
    )
    # persist immediately
    _save_checkpoint(config, {"phase": "setup_complete"})
    return config
```

### Success Criteria for Phase 0 → Phase 1 Transition

- `root_dir` exists and `os.access(root_dir, os.W_OK)` is True
- All subdirs in `WORKSPACE_SUBDIRS` exist
- `checkpoint_path` parent directory exists and is writable
- `WorkspaceConfig` serialized to `checkpoints/pipeline_state.json`

---

## 4. Phase 1: Planning & Design with Approval Gate

### State Machine

```
[ENTER PHASE 1]
      |
      v
 [PlannerAgent]
  Input: research_topic, workspace_config, optional user_feedback
      |
   [LLM call, max PLAN_MAX_RETRIES=3 on LLM error]
      |
      +--[success]--> PlannerResult
      |
      +--[fail × PLAN_MAX_RETRIES]--> PIPELINE ERROR (unrecoverable)
      |
      v
 [DesignerAgent]
  Input: PlannerResult
      |
   [LLM call, max DESIGN_MAX_RETRIES=3 on LLM error]
      |
      +--[success]--> DesignerResult
      |
      +--[fail × DESIGN_MAX_RETRIES]--> PIPELINE ERROR
      |
      v
 [APPROVAL GATE]  <---------------------------+
  Display: PlannerResult + DesignerResult      |
  Block: wait for user response                |
      |                                        |
      +--[APPROVE]----> Phase 2 (coding)       |
      |                                        |
      +--[REJECT]------> re-run PlannerAgent --+
      |  inject: user_feedback                 |
      |                                        |
      +--[MODIFY]------> re-run PlannerAgent --+
         inject: user_edits merged into        |
                 original PlannerResult        |
```

### Approval Gate Detail

The approval gate is a synchronous blocking call. Before blocking, pipeline state is checkpointed so the system can resume after restart.

```python
from enum import Enum

class ApprovalDecision(Enum):
    APPROVE = "approve"
    REJECT  = "reject"
    MODIFY  = "modify"

@dataclass
class ApprovalResponse:
    decision:  ApprovalDecision
    feedback:  str | None   # populated on REJECT or MODIFY
    edits:     dict | None  # populated on MODIFY: partial override of PlannerResult fields

def run_approval_gate(
    planner_result: PlannerResult,
    designer_result: DesignerResult,
    notifier: UserNotifier,
    checkpoint_fn: callable,
    timeout_seconds: float = APPROVAL_TIMEOUT_SECONDS,  # default: 3600 (1 hour)
) -> ApprovalResponse:

    # save state so we can resume if the process restarts
    checkpoint_fn({
        "phase": "waiting_approval",
        "planner_result": planner_result.model_dump(),
        "designer_result": designer_result.model_dump(),
    })

    # format display for user
    display = _format_approval_display(planner_result, designer_result)
    notifier.show_approval_request(display)

    # blocking wait
    response = notifier.wait_for_approval(timeout_seconds=timeout_seconds)

    if response is None:
        # timeout: treat as APPROVE to avoid blocking forever
        # log a warning
        return ApprovalResponse(decision=ApprovalDecision.APPROVE, feedback="[TIMEOUT: auto-approved]", edits=None)

    return response
```

### Display Format for Approval Gate

```
============================================================
PLAN & DESIGN REVIEW — Run ID: {run_id}
============================================================

RESEARCH PLAN
  Problem Statement : {problem_statement}
  Research Questions: {research_questions}
  Hypotheses        : {hypotheses}
  Success Criteria  : {success_criteria}
  Baseline Comparisons: {baseline_comparisons}
  Risk Items        : {risk_items}

DESIGN
  Files to generate : {len(file_tree)} files across 3 stages
  Stage 1 (foundations): {stage1_files}
  Stage 2 (core logic) : {stage2_files}
  Stage 3 (entry/glue) : {stage3_files}
  Entry point       : {entry_point}

============================================================
Enter decision:
  [A] APPROVE — proceed to coding
  [R] REJECT  — restart planning (provide feedback)
  [M] MODIFY  — keep plan, adjust specific fields
============================================================
```

### Loop Conditions for Phase 1

- Maximum `APPROVAL_LOOP_MAX` (default: 10) reject/modify cycles before forcing user to explicitly override with `FORCE_APPROVE`
- Each loop injects the user's feedback into PlannerAgent's context via `user_feedback_history` list (all prior feedbacks preserved)
- DesignerAgent always re-runs after PlannerAgent re-runs (design must reflect new plan)

---

## 5. Phase 2: Staged Coding

### Stage Assignment Logic

DesignerAgent assigns each file to a stage (1, 2, or 3) based on the dependency graph. Stage 1 files have no internal dependencies; stage 3 files are entry points and integration scripts.

```
Stage 1: leaf nodes in import graph (config, constants, types, base classes)
Stage 2: internal nodes (business logic, algorithms, data loaders)
Stage 3: roots of import graph (main.py, entry points, test runners)
```

### Top-Level Stage Loop State Machine

```
[ENTER PHASE 2]
  stage_results = {}   # stage_id -> StageResult
      |
      v
 FOR stage IN [1, 2, 3]:
      |
      v
 +----+---------------------------+
 |  CoderAgent writes all files   |
 |  in this stage                 |
 +----+---------------------------+
      |
      v
 FOR each file F in stage.files:
      |
      v
 +----+-----------------------------+
 |  SyntaxChecker(F)                |
 |  ImportChecker(F)                |
 +----+-----------------------------+
      |
      +--[both pass]--> record OK, continue to next file
      |
      +--[fail]--> FILE REPAIR LOOP (see below)
      |
 [all files in stage processed]
      |
      v
 [Stage Integration Check]
  run: python -c "import {all_stage_modules}"
      |
      +--[pass]--> stage_results[stage] = StageResult(ok=True)
      |            continue to next stage
      |
      +--[fail]--> FILE REPAIR LOOP (entry = stage integration error)
```

### File Repair Loop — NO Circuit Breaker

```
FILE REPAIR LOOP for file F, error E:
  attempt_count = 0
  fix_history = []      # list of (attempt_number, error_message, repair_description)

  WHILE True:           # ← NO circuit breaker. Never silently give up.
    attempt_count += 1

    IF attempt_count <= MAX_AUTO_ATTEMPTS:   # default: 5
      CoderAgent.repair(F, E, fix_history)
      result = run_checks(F)

      IF result.ok:
        fix_history.append((attempt_count, str(E), "auto-repair succeeded"))
        BREAK   # repair succeeded

      ELSE:
        E = result.error
        fix_history.append((attempt_count, str(E), "auto-repair failed"))
        continue WHILE

    ELSE:
      # AUTO ATTEMPTS EXHAUSTED — escalate to user
      # This path is NOT a failure. It is a state transition.
      user_response = escalate_to_user(
          file_path    = F,
          error        = E,
          attempt_count = attempt_count,
          fix_history  = fix_history,
      )

      IF user_response.action == "continue":
          # user grants more budget
          MAX_AUTO_ATTEMPTS += user_response.extra_attempts   # default extra: 3
          continue WHILE   # re-enter loop with increased budget

      ELIF user_response.action == "skip":
          # stub the file: write a minimal valid Python file
          write_stub(F)
          mark_as_stubbed(F)
          BREAK   # continue pipeline with stub

      ELIF user_response.action == "provide_fix":
          # user gives a natural language hint
          inject_hint_into_coder(F, user_response.hint)
          MAX_AUTO_ATTEMPTS += 3   # extra budget for hint-guided retry
          attempt_count -= 1       # don't count this as an attempt
          continue WHILE

      ELIF user_response.action == "manual_edit":
          # user edits the file directly in their editor
          wait_for_user_signal("edited")   # user signals "done editing"
          result = run_checks(F)
          IF result.ok:
              BREAK
          ELSE:
              E = result.error
              # re-enter loop; user may edit again or choose different action
              continue WHILE
```

### Python State Machine for File Repair Loop

```python
from dataclasses import dataclass, field
from enum import Enum
import time

class UserAction(Enum):
    CONTINUE     = "continue"
    SKIP         = "skip"
    PROVIDE_FIX  = "provide_fix"
    MANUAL_EDIT  = "manual_edit"

@dataclass
class RepairLoopState:
    file_path:      str
    stage:          int
    attempt_count:  int = 0
    max_auto:       int = 5          # MAX_AUTO_ATTEMPTS, mutable per user grant
    fix_history:    list = field(default_factory=list)
    stubbed:        bool = False
    resolved:       bool = False
    started_at:     float = field(default_factory=time.time)

def run_repair_loop(
    state: RepairLoopState,
    coder_agent,
    checker,
    escalator,
    checkpoint_fn,
) -> RepairLoopState:

    while True:
        state.attempt_count += 1

        if state.attempt_count <= state.max_auto:
            error = coder_agent.repair(state.file_path, state.fix_history)
            result = checker.check(state.file_path)

            if result.ok:
                state.resolved = True
                state.fix_history.append({
                    "attempt": state.attempt_count,
                    "action": "auto_repair",
                    "outcome": "success",
                })
                checkpoint_fn(state)
                return state

            state.fix_history.append({
                "attempt": state.attempt_count,
                "action": "auto_repair",
                "outcome": "failed",
                "error": str(result.error),
            })
            checkpoint_fn(state)
            continue

        # escalate
        checkpoint_fn(state)   # save before blocking
        user_resp = escalator.escalate(state)

        if user_resp.action == UserAction.CONTINUE:
            state.max_auto += user_resp.extra_attempts
            # attempt_count already incremented; will be <= max_auto on next iteration
            continue

        elif user_resp.action == UserAction.SKIP:
            _write_stub(state.file_path)
            state.stubbed = True
            state.resolved = True   # "resolved" as stub
            checkpoint_fn(state)
            return state

        elif user_resp.action == UserAction.PROVIDE_FIX:
            coder_agent.inject_hint(state.file_path, user_resp.hint)
            state.max_auto += 3
            state.attempt_count -= 1   # hint-guided; don't penalize count
            continue

        elif user_resp.action == UserAction.MANUAL_EDIT:
            escalator.wait_for_user_edit_signal(state.file_path)
            result = checker.check(state.file_path)
            if result.ok:
                state.resolved = True
                checkpoint_fn(state)
                return state
            # failed even after manual edit — loop continues, user may act again
            state.fix_history.append({
                "attempt": state.attempt_count,
                "action": "manual_edit",
                "outcome": "failed",
                "error": str(result.error),
            })
            state.attempt_count -= 1   # manual edit does not consume auto budget
            continue
```

### Smoke Test After All 3 Stages

```
[ALL STAGES COMPLETE]
      |
      v
 [Smoke Test]
  cmd = f"python -c 'import {entry_point_module}'"
  OR
  cmd = f"python {entry_point} --smoke-test"
      |
      +--[returncode == 0]--> Phase 3 (execution)
      |
      +--[returncode != 0]--> SMOKE TEST REPAIR LOOP
                               (same escalation pattern as file repair)
                               escalation target: the failing import chain
```

---

## 6. Phase 3: Experiment Execution

### State Machine

```
[ENTER PHASE 3]
      |
      v
 +----+-----------------------------+
 |  ExecutorAgent                   |
 |  cmd: python {entry_point}       |
 |  cwd: workspace_config.root_dir  |
 |  timeout: EXEC_TIMEOUT_SECONDS   |
 +----+-----------------------------+
      |
      v
 [Capture stdout, stderr, return_code, duration]
      |
      +--[return_code == 0 AND result.json exists]
      |      |
      |      v
      |  Collect artifacts (plots, logs, metrics)
      |  Build ExecutorResult
      |      |
      |      v
      |  Phase 4 (writing)
      |
      +--[return_code != 0 OR result.json missing]
             |
             v
        EXECUTION REPAIR LOOP  (same structure as file repair, NO circuit breaker)
             |
             attempt_count += 1
             |
             IF attempt_count <= MAX_EXEC_ATTEMPTS (default: 3):
               ExecutorAgent diagnoses stderr
               CoderAgent patches source files based on diagnosis
               Re-run experiment
             ELSE:
               escalate_to_user(...)
               handle user response (continue/skip/provide_fix/manual_edit)
```

### ExecutorResult Schema

```python
@dataclass
class ExecutorResult:
    success:        bool
    return_code:    int
    duration_sec:   float
    stdout_tail:    str      # last 2000 chars of stdout
    stderr_tail:    str      # last 2000 chars of stderr
    result_json:    dict | None      # parsed results/result.json, or None
    artifact_paths: list[str]        # all generated files (plots, logs, etc.)
    metrics:        dict             # key metrics extracted from result.json
    repair_count:   int              # how many repairs were needed
    stubbed_files:  list[str]        # files that were stubbed during Phase 2
```

---

## 7. Phase 4: Section-by-Section Paper Writing

### Section Order and Rationale

```
Section write order:
  1. Experiments      ← depends on ExecutorResult, written first to anchor facts
  2. Proposed_Method  ← describes what was implemented
  3. Introduction     ← motivates the work
  4. Related_Works    ← positions the work
  5. Conclusion       ← summarizes findings
  6. References       ← bibliography
  7. Abstract         ← written last (summarizes the whole paper)
```

Note: The section order for writing differs from the section order in the final paper. The writing order is dependency-topological (facts first, framing last).

### State Machine

```
[ENTER PHASE 4]
  section_results = {}   # section_name -> SectionResult
  context_bank = {}      # section_name -> written_text  (coherence anchor)
      |
      v
 FOR section IN WRITE_ORDER:
      |
      v
 +----+-----------------------------+
 |  WriterAgent writes section      |
 |  Input: ExecutorResult,          |
 |         context_bank (prior      |
 |         written sections),       |
 |         section-specific prompt  |
 +----+-----------------------------+
      |
      v
 +----+-----------------------------+
 |  SelfVerifier checks:            |
 |  1. MIN_WORDS satisfied?         |
 |  2. Citations valid format?      |
 |  3. LaTeX/Markdown syntax?       |
 |  4. References actual results?   |
 +----+-----------------------------+
      |
      +--[all checks pass]--------> context_bank[section] = text
      |                             section_results[section] = SectionResult(ok=True)
      |                             continue to next section
      |
      +--[check fails, revision < MAX_SECTION_REVISIONS (default: 3)]:
      |        WriterAgent revises with verifier feedback
      |        re-run SelfVerifier
      |        (loop up to MAX_SECTION_REVISIONS)
      |
      +--[check fails, revision == MAX_SECTION_REVISIONS]:
               mark section as NEEDS_REVIEW
               context_bank[section] = text   # use current text for coherence
               section_results[section] = SectionResult(ok=False, needs_review=True)
               continue to next section  (do NOT block pipeline)

 [ALL SECTIONS WRITTEN]
      |
      v
 [IntegrationChecker]
  Verify cross-section coherence:
  - Do method descriptions in Proposed_Method match experiments in Experiments?
  - Does Conclusion cite actual result numbers?
  - Does Introduction's claimed contribution appear in Proposed_Method?
      |
      v
 [FinalQuality Score]
  score = weighted average of per-section quality scores
      |
      +--[score >= QUALITY_THRESHOLD (default: 0.75)]
      |      |
      |      v
      |  Write paper to disk (paper.md or paper.tex)
      |  Done
      |
      +--[score < QUALITY_THRESHOLD]
             |
             v
        targeted re-write of sections with score < SECTION_REWRITE_THRESHOLD (default: 0.60)
        (same per-section loop, max MAX_SECTION_REVISIONS additional revisions)
             |
             v
        Final paper written regardless of score
        Low-score sections flagged in paper frontmatter
```

### Context Bank — Coherence Mechanism

Each written section is stored in `context_bank`. When writing a subsequent section, the WriterAgent receives a compressed summary of all prior sections:

```python
def build_context_summary(context_bank: dict[str, str], current_section: str) -> str:
    """
    Provides coherence anchors without consuming full token budget.
    Returns at most CONTEXT_BANK_MAX_CHARS (default: 3000) chars total.
    """
    prior = {k: v for k, v in context_bank.items() if k != current_section}
    summaries = []
    budget = CONTEXT_BANK_MAX_CHARS
    for section_name, text in prior.items():
        summary = f"[{section_name}] {text[:200]}..."   # first 200 chars
        summaries.append(summary)
        budget -= len(summary)
        if budget <= 0:
            break
    return "\n".join(summaries)
```

### SelfVerifier Checks (Detailed)

```python
import re

class SelfVerifier:

    # Check 1: minimum word count
    def check_min_words(self, text: str, section: str) -> VerifyResult:
        MIN_WORDS = {
            "Abstract":         150,
            "Introduction":     400,
            "Related_Works":    400,
            "Proposed_Method":  500,
            "Experiments":      600,
            "Conclusion":       200,
            "References":       0,    # checked by citation format instead
        }
        count = len(text.split())
        threshold = MIN_WORDS.get(section, 200)
        if count < threshold:
            return VerifyResult(ok=False, message=f"Too short: {count} words, need {threshold}")
        return VerifyResult(ok=True)

    # Check 2: citation format
    # Supports both [1] and \cite{key} and (Author, Year) styles
    CITATION_PATTERNS = [
        r"\[\d+\]",                          # [1], [12]
        r"\\cite\{[a-zA-Z0-9_:,\s]+\}",     # \cite{key}
        r"\([A-Z][a-z]+(?:\s+et\s+al\.)?,\s*\d{4}\)",  # (Author et al., 2023)
    ]

    def check_citations(self, text: str, section: str) -> VerifyResult:
        if section == "References":
            # References section: each line should be a valid citation entry
            lines = [l.strip() for l in text.split("\n") if l.strip()]
            valid = [l for l in lines if re.match(r"^\[\d+\]|^\d+\.|^-\s", l)]
            if len(valid) < MIN_REFERENCE_ENTRIES:   # default: 5
                return VerifyResult(ok=False, message=f"Too few reference entries: {len(valid)}")
        else:
            # Other sections: at least one citation if it mentions prior work
            mentions_prior = bool(re.search(
                r"\b(previous|prior|existing|proposed by|et al|shown in|according to)\b",
                text, re.IGNORECASE
            ))
            if mentions_prior:
                has_citation = any(re.search(p, text) for p in self.CITATION_PATTERNS)
                if not has_citation:
                    return VerifyResult(ok=False, message="Prior work mentioned but no citation found")
        return VerifyResult(ok=True)

    # Check 3: references actual experimental results
    def check_references_results(self, text: str, executor_result: ExecutorResult) -> VerifyResult:
        if executor_result is None or not executor_result.metrics:
            return VerifyResult(ok=True)   # no results to check against
        # extract numeric values from text
        text_numbers = set(re.findall(r"\b\d+\.\d+\b|\b\d{2,}\b", text))
        # extract numbers from metrics
        metric_numbers = set()
        for v in executor_result.metrics.values():
            if isinstance(v, (int, float)):
                metric_numbers.add(str(round(float(v), 3)))
                metric_numbers.add(str(int(v)))
        overlap = text_numbers & metric_numbers
        # Experiments and Conclusion should cite at least one real number
        if not overlap and text_numbers:
            return VerifyResult(
                ok=False,
                message=f"No actual result numbers found in text. Available metrics: {executor_result.metrics}"
            )
        return VerifyResult(ok=True)

    # Check 4: LaTeX/Markdown syntax
    LATEX_UNCLOSED = [
        (r"\\begin\{(\w+)\}", r"\\end\{\1\}"),   # \begin{X} must have \end{X}
    ]
    MARKDOWN_UNCLOSED = [
        (r"```", "code fence"),
    ]

    def check_syntax(self, text: str, fmt: Literal["markdown","latex"]) -> VerifyResult:
        if fmt == "latex":
            for open_pat, close_pat in self.LATEX_UNCLOSED:
                opens  = re.findall(open_pat, text)
                closes = re.findall(close_pat, text)
                for env in opens:
                    if opens.count(env) != closes.count(env):
                        return VerifyResult(ok=False, message=f"Unclosed LaTeX environment: {env}")
        elif fmt == "markdown":
            fence_count = text.count("```")
            if fence_count % 2 != 0:
                return VerifyResult(ok=False, message="Unclosed code fence in Markdown")
        return VerifyResult(ok=True)
```

---

## 8. Cross-Phase State Persistence

All state is checkpointed to `{root_dir}/checkpoints/pipeline_state.json` before any blocking operation. The JSON schema:

```json
{
  "run_id": "abc12345",
  "session_id": "uuid",
  "phase": "coding",
  "sub_phase": "stage_2_file_repair",
  "paused_for_user": true,
  "workspace": {
    "root_dir": "/path/to/workspace",
    "checkpoint_path": "/path/to/checkpoints/pipeline_state.json"
  },
  "planner_result": { "...": "..." },
  "designer_result": { "...": "..." },
  "coding_state": {
    "current_stage": 2,
    "completed_stages": [1],
    "file_repair_states": {
      "src/model.py": {
        "attempt_count": 6,
        "max_auto": 5,
        "stubbed": false,
        "resolved": false,
        "fix_history": []
      }
    },
    "stubbed_files": []
  },
  "executor_result": null,
  "paper_state": {
    "completed_sections": [],
    "needs_review_sections": [],
    "context_bank": {}
  }
}
```

### Resume Logic

```python
def resume_from_checkpoint(checkpoint_path: str) -> PipelineState:
    state = load_json(checkpoint_path)
    phase = state["phase"]

    if state["paused_for_user"]:
        # re-display the escalation context to user
        # user must respond before pipeline continues
        pass

    if phase == "coding":
        # resume at the specific file that was being repaired
        resume_coding(state["coding_state"])
    elif phase == "waiting_approval":
        # re-display approval request
        resume_approval(state["planner_result"], state["designer_result"])
    # ... etc.
```

---

## 9. User Interaction Protocol

### Escalation Message Format (for file repair)

```
============================================================
REPAIR ESCALATION — Requires Your Input
Run ID: {run_id}  |  Stage: {stage}  |  File: {file_path}
============================================================

PROBLEM
  Error type   : {error_type}  (e.g. SyntaxError, ImportError)
  Error message: {error_message}
  Location     : {file_path}:{line_number}

REPAIR HISTORY ({attempt_count} auto-attempts failed)
  Attempt 1: {description_of_attempt_1}
    -> Error: {resulting_error_1}
  Attempt 2: {description_of_attempt_2}
    -> Error: {resulting_error_2}
  ...

OPTIONS
  [C] CONTINUE    — grant {DEFAULT_EXTRA_ATTEMPTS} more auto-repair attempts
  [S] SKIP        — replace file with a minimal stub and continue
  [H] HINT        — provide a natural language fix hint to the coder
  [E] EDIT        — edit the file manually; type DONE when finished

Enter choice (C/S/H/E):
============================================================
```

### User Input Handling (CLI)

```python
def wait_for_escalation_response(context: EscalationContext) -> UserResponse:
    print(format_escalation_message(context))

    while True:
        choice = input("Enter choice (C/S/H/E): ").strip().upper()

        if choice == "C":
            extra = input(f"Extra attempts to grant [default: {DEFAULT_EXTRA_ATTEMPTS}]: ").strip()
            extra = int(extra) if extra.isdigit() else DEFAULT_EXTRA_ATTEMPTS
            return UserResponse(action=UserAction.CONTINUE, extra_attempts=extra)

        elif choice == "S":
            confirm = input("Skip file and replace with stub? [y/N]: ").strip().lower()
            if confirm == "y":
                return UserResponse(action=UserAction.SKIP)

        elif choice == "H":
            hint = input("Enter fix hint: ").strip()
            if hint:
                return UserResponse(action=UserAction.PROVIDE_FIX, hint=hint)

        elif choice == "E":
            print(f"Edit the file at: {context.file_path}")
            print("Type DONE and press Enter when finished.")
            while input().strip().upper() != "DONE":
                pass
            return UserResponse(action=UserAction.MANUAL_EDIT)

        else:
            print("Invalid choice. Enter C, S, H, or E.")
```

---

## 10. Phase Transition Criteria

| From Phase | To Phase | Condition |
|------------|----------|-----------|
| 0 (setup)    | 1 (planning) | workspace writable, dirs created, checkpoint saved |
| 1 planning   | 1 design     | PlannerResult parsed, all required fields non-empty |
| 1 design     | 1 approval   | DesignerResult parsed, file_tree non-empty, entry_point valid |
| 1 approval   | 2 coding     | User decision == APPROVE |
| 1 approval   | 1 planning   | User decision == REJECT or MODIFY |
| 2 coding     | 3 execution  | All stages complete (some files may be stubs), smoke test passes |
| 3 execution  | 4 writing    | return_code == 0 AND result.json exists (or user chose skip) |
| 4 writing    | done         | All 7 sections written (some may be NEEDS_REVIEW), paper.md exists |

No phase transition is allowed if `paused_for_user == True`. The pipeline is blocked until user responds.
