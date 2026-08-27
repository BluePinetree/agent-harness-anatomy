# MODULE_SPEC.md — Research Automation System V4

---

## 1. Directory Structure

```
crewai_prototype/
│
├── api/                              # HTTP layer
│   ├── __init__.py
│   ├── app.py                        # FastAPI factory (updated)
│   ├── schemas.py                    # Pydantic request/response models (updated)
│   └── routes/
│       ├── __init__.py
│       ├── runs.py                   # POST /runs, GET status, DELETE (NEW — replaces research.py)
│       ├── approval.py               # POST /approve, POST /guidance (NEW)
│       ├── stream.py                 # GET /stream SSE (NEW — extracted from research.py)
│       ├── artifacts.py              # unchanged
│       ├── sessions.py               # unchanged
│       └── contract.py               # unchanged
│
├── orchestration/                    # Pipeline control (partially NEW)
│   ├── __init__.py
│   ├── pipeline_orchestrator.py      # NEW — replaces research_coordinator_v3.py
│   ├── approval_gate.py              # NEW
│   ├── guidance_registry.py          # NEW
│   ├── cancellation_token.py         # NEW
│   ├── input_normalizer.py           # unchanged
│   └── input_enricher.py             # unchanged
│
├── phases/                           # NEW — one module per pipeline phase
│   ├── __init__.py
│   ├── phase0_workspace.py
│   ├── phase1_planning.py
│   ├── phase2_coding.py
│   ├── phase3_execution.py
│   └── phase4_writing.py
│
├── core/                             # Shared infrastructure (REUSED)
│   ├── __init__.py
│   ├── handoff_models.py             # extended with new models (see API_SPEC.md §4)
│   ├── handoff_store.py              # unchanged
│   ├── llm_factory.py                # unchanged
│   ├── llm_retry.py                  # unchanged
│   ├── logger.py                     # unchanged
│   ├── config.py                     # unchanged
│   ├── json_extractor.py             # unchanged
│   └── project_manifest.py          # unchanged
│
├── crew_agents/                      # CrewAI agent factories (mostly REUSED)
│   ├── __init__.py
│   ├── planner.py                    # unchanged
│   ├── designer.py                   # unchanged
│   ├── file_coder.py                 # updated: add SyntaxCheckTool, ImportCheckTool
│   ├── executor.py                   # unchanged
│   ├── analyzer.py                   # unchanged
│   └── writer.py                     # updated: WriteReportSectionTool
│
├── crew_tools/                       # CrewAI tool implementations
│   ├── __init__.py
│   ├── workspace_tools.py            # unchanged
│   ├── execution_tools.py            # unchanged
│   ├── edit_tool.py                  # unchanged
│   ├── report_tools.py               # unchanged
│   └── syntax_tools.py               # NEW: SyntaxCheckTool, ImportCheckTool
│
├── runtime/                          # Event & session persistence (REUSED)
│   ├── __init__.py
│   ├── event_store.py                # updated: new event types registered
│   ├── session_store.py              # unchanged
│   ├── models.py                     # updated: new EVENT_TYPES, new status values
│   ├── run_repository.py             # unchanged
│   ├── state_calculator.py           # updated: new status rendering
│   └── stream_service.py             # unchanged
│
├── scaffolds/                        # Scaffold profiles (unchanged)
├── workspace/                        # Workspace service helpers (unchanged)
├── outputs/                          # Runtime output directory (git-ignored)
└── tests/
    ├── test_approval_gate.py         # NEW
    ├── test_guidance_registry.py     # NEW
    ├── test_staged_coder.py          # NEW
    ├── test_phase1_planning.py       # NEW
    └── ...
```

---

## 2. Module Specifications

---

### 2.1 `orchestration/approval_gate.py`

**Purpose:** Thread-safe registry of per-run plan approval gates. Allows the pipeline thread to block until the user approves or rejects the plan via HTTP.

**Dependencies:** `threading`, `time`, `dataclasses`, `core/handoff_models.py`

```python
from __future__ import annotations

import threading
import time
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class ApprovalGateState:
    """
    Mutable state for one plan approval gate.

    Fields:
        plan_payload (dict): The full plan to show the user. Set when the gate opens.
        _event (threading.Event): Internal event. Callers use wait() / resolve().
        action (str | None): "approve" or "reject". Set by resolve().
        feedback (str | None): User revision text. Set when action="reject".
        created_at (float): Unix timestamp when gate opened.
    """

    plan_payload: dict
    _event: threading.Event = field(default_factory=threading.Event, init=False)
    action: Optional[str] = field(default=None, init=False)
    feedback: Optional[str] = field(default=None, init=False)
    created_at: float = field(default_factory=time.time, init=False)

    def wait(self, timeout_seconds: float = 3600.0) -> bool:
        """
        Block the calling thread until resolve() is called or timeout expires.

        Returns:
            True if resolved by user before timeout.
            False if timeout expired (caller should auto-approve with warning).
        """

    def resolve(self, action: str, feedback: Optional[str] = None) -> None:
        """
        Set action and feedback, then unblock the waiting thread.

        Args:
            action: "approve" or "reject"
            feedback: Required text when action="reject"

        Raises:
            ValueError: If gate was already resolved.
        """

    @property
    def is_resolved(self) -> bool:
        """Return True if resolve() has been called."""


class ApprovalRegistry:
    """
    Thread-safe dict mapping run_id -> ApprovalGateState.

    Lifecycle:
        1. orchestrator calls open(run_id, plan_payload) when Phase 1 completes
        2. API route calls resolve(run_id, action, feedback) on POST /approve
        3. orchestrator's wait() returns; gate is automatically closed after resolution
    """

    def __init__(self) -> None:
        self._lock: threading.Lock
        self._gates: dict[str, ApprovalGateState]

    def open(self, run_id: str, plan_payload: dict) -> ApprovalGateState:
        """
        Create and register a gate for the given run.

        Raises:
            RuntimeError: If a gate already exists for run_id.
        """

    def resolve(self, run_id: str, action: str, feedback: Optional[str] = None) -> None:
        """
        Deliver the user's decision to the waiting pipeline thread.

        Raises:
            KeyError: If no gate exists for run_id.
            ValueError: If gate already resolved.
        """

    def close(self, run_id: str) -> None:
        """Remove the gate entry (call after wait() returns)."""

    def get(self, run_id: str) -> Optional[ApprovalGateState]:
        """Return the gate state, or None if not present."""

    def is_pending(self, run_id: str) -> bool:
        """Return True if a gate exists and has not been resolved."""
```

---

### 2.2 `orchestration/guidance_registry.py`

**Purpose:** Thread-safe registry of per-run user guidance requests. Allows the repair loop to block until the user submits guidance via HTTP.

**Dependencies:** `threading`, `time`, `dataclasses`

```python
from __future__ import annotations

import threading
import time
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class GuidanceRequestState:
    """
    One pending guidance request for a stuck repair loop.

    Fields:
        file_path (str): The file that is failing.
        error_summary (str): Human-readable summary of the last error.
        repair_attempts (int): How many LLM repairs were attempted before asking.
        last_error_output (str): Truncated stderr/traceback from the last attempt.
        guidance (str | None): The user's response. Set by deliver().
        _event (threading.Event): Unblocked when guidance arrives.
        created_at (float): Unix timestamp.
    """

    file_path: str
    error_summary: str
    repair_attempts: int
    last_error_output: str
    _event: threading.Event = field(default_factory=threading.Event, init=False)
    guidance: Optional[str] = field(default=None, init=False)
    created_at: float = field(default_factory=time.time, init=False)

    def wait(self, timeout_seconds: float = 7200.0) -> bool:
        """
        Block until deliver() is called or timeout expires.

        Returns:
            True if guidance arrived before timeout.
            False if timeout expired (caller should update status to "paused").
        """

    def deliver(self, guidance: str) -> None:
        """
        Store user guidance and unblock the waiting thread.

        Raises:
            ValueError: If guidance already delivered.
        """

    @property
    def is_delivered(self) -> bool: ...


class GuidanceRegistry:
    """
    Thread-safe dict mapping run_id -> GuidanceRequestState.

    Only one guidance request may be open per run at a time.
    """

    def __init__(self) -> None:
        self._lock: threading.Lock
        self._requests: dict[str, GuidanceRequestState]

    def open(
        self,
        run_id: str,
        file_path: str,
        error_summary: str,
        repair_attempts: int,
        last_error_output: str,
    ) -> GuidanceRequestState:
        """
        Create a guidance request for the given run.

        Raises:
            RuntimeError: If a guidance request is already open for run_id.
        """

    def deliver(self, run_id: str, guidance: str) -> None:
        """
        Deliver guidance from the API layer to the pipeline thread.

        Raises:
            KeyError: No request for run_id.
            ValueError: Already delivered.
        """

    def close(self, run_id: str) -> None:
        """Remove the request entry."""

    def get(self, run_id: str) -> Optional[GuidanceRequestState]:
        """Return the request state, or None."""

    def is_pending(self, run_id: str) -> bool:
        """Return True if a request exists and guidance not yet delivered."""
```

---

### 2.3 `orchestration/cancellation_token.py`

**Purpose:** Lightweight cooperative cancellation flag that the pipeline thread checks at every phase boundary and within repair loops.

```python
from __future__ import annotations

import threading


class CancellationToken:
    """
    A thread-safe flag that any component can set to signal cancellation.

    Usage in pipeline:
        token = CancellationToken()
        token.check()  # raises CancelledError if set
    """

    def __init__(self) -> None:
        self._cancelled: bool = False
        self._lock: threading.Lock

    def cancel(self) -> None:
        """Set the cancellation flag. Idempotent."""

    def check(self) -> None:
        """
        Raise CancelledError if cancel() has been called.

        Call at: phase transitions, top of each repair loop iteration,
        and before each LLM call.
        """

    @property
    def is_cancelled(self) -> bool: ...


class CancellationRegistry:
    """
    Thread-safe dict mapping run_id -> CancellationToken.
    Managed by PipelineOrchestrator.
    """

    def create(self, run_id: str) -> CancellationToken: ...
    def cancel(self, run_id: str) -> None: ...
    def remove(self, run_id: str) -> None: ...
    def get(self, run_id: str) -> Optional[CancellationToken]: ...
```

---

### 2.4 `orchestration/pipeline_orchestrator.py`

**Purpose:** Top-level run lifecycle manager. Replaces `crew/research_coordinator_v3.py`. Owns the thread-per-run model, phase delegation, and cross-cutting concerns (status updates, cost tracking, event emission).

**Dependencies:** `phases/`, `orchestration/approval_gate.py`, `orchestration/guidance_registry.py`, `orchestration/cancellation_token.py`, `runtime/`, `core/`

```python
from __future__ import annotations

import threading
import traceback
from dataclasses import dataclass
from pathlib import Path
from typing import Any

from crewai import LLM

from core.handoff_models import PlanBundle, CodingResult, ExecutionResult, WritingResult
from core.llm_retry import get_cost_tracker
from orchestration.approval_gate import ApprovalRegistry
from orchestration.guidance_registry import GuidanceRegistry
from orchestration.cancellation_token import CancellationRegistry, CancellationToken
from runtime.event_store import EventStore
from runtime.models import RunSession
from runtime.session_store import SessionStore


@dataclass
class PreparedRun:
    """
    Bundle produced by prepare_run(), passed to launch_prepared().

    Fields:
        session (RunSession): Persisted session metadata.
        orchestrator_input (dict): Normalized input from API request.
        output_path (Path): Resolved output root for this run.
    """

    session: RunSession
    orchestrator_input: dict[str, Any]
    output_path: Path


class PipelineOrchestrator:
    """
    Manages the lifecycle of research pipeline runs.

    One instance is shared across all concurrent runs (via app.state.services).
    Each run gets its own daemon thread, ApprovalGate, GuidanceRequest, and
    CancellationToken.

    Thread safety:
        - _launch_lock: protects _active_threads dict
        - ApprovalRegistry, GuidanceRegistry, CancellationRegistry are internally locked
        - EventStore and SessionStore use atomic file writes

    Constructor Args:
        session_store (SessionStore): Persists session metadata.
        event_store (EventStore): Persists pipeline events as JSONL.
        output_root (Path): Default parent directory for run outputs.
        llm (LLM): Shared base LLM instance (per-agent LLMs are derived from it).
        approval_registry (ApprovalRegistry): Injected; shared with API routes.
        guidance_registry (GuidanceRegistry): Injected; shared with API routes.
        cancellation_registry (CancellationRegistry): Injected; shared with API routes.
    """

    def __init__(
        self,
        session_store: SessionStore,
        event_store: EventStore,
        output_root: Path,
        llm: LLM,
        approval_registry: ApprovalRegistry,
        guidance_registry: GuidanceRegistry,
        cancellation_registry: CancellationRegistry,
    ) -> None: ...

    # ------------------------------------------------------------------
    # Public interface (called from API routes)
    # ------------------------------------------------------------------

    def prepare_run(self, request_dict: dict[str, Any]) -> PreparedRun:
        """
        Normalize input, generate run_id, persist initial session, return PreparedRun.

        Does NOT start the pipeline thread.

        Returns:
            PreparedRun with status="queued"
        """

    def launch_prepared(self, prepared: PreparedRun) -> None:
        """
        Start the pipeline thread for the prepared run.

        Raises:
            RuntimeError: If a thread is already active for this run_id.
        """

    def cancel(self, run_id: str) -> None:
        """
        Signal the pipeline thread to cancel at its next check point.

        Raises:
            KeyError: Unknown run_id.
            RuntimeError: Run already in terminal state.
        """

    def approve_plan(self, run_id: str, action: str, feedback: str | None = None) -> None:
        """
        Deliver plan approval/rejection to the pipeline thread.

        Raises:
            KeyError: No pending approval gate for run_id.
            ValueError: Invalid action, or rejection without feedback.
        """

    def deliver_guidance(self, run_id: str, guidance: str) -> None:
        """
        Deliver user guidance to a stuck repair loop.

        Raises:
            KeyError: No pending guidance request for run_id.
        """

    # ------------------------------------------------------------------
    # Internal — pipeline thread
    # ------------------------------------------------------------------

    def _run_thread(self, prepared: PreparedRun) -> None:
        """
        Entry point for the daemon thread.
        Calls _execute() and handles fatal exceptions:
            - Emits SYSTEM_END with status=failed
            - Updates SessionStore
            - Removes thread from _active_threads
        """

    def _execute(self, prepared: PreparedRun) -> None:
        """
        Main pipeline execution sequence:

        1. _phase0_workspace_setup(prepared) -> WorkspaceSetupResult
        2. loop:
               _phase1_planning(prepared, ws_result) -> PlanBundle
               gate = approval_registry.open(run_id, plan_bundle.dict())
               emit PLAN_AWAITING_APPROVAL
               update_status("awaiting_approval")
               resolved = gate.wait(timeout=approval_timeout)
               if not resolved: auto-approve, emit warning
               if gate.action == "reject":
                   inject feedback, continue loop (re-run phase1)
               break
        3. _phase2_coding(prepared, plan_bundle) -> CodingResult
        4. _phase3_execution(prepared, coding_result) -> ExecutionResult
        5. _phase4_writing(prepared, plan_bundle, execution_result) -> WritingResult
        6. update_status("completed"), emit SYSTEM_END

        CancellationToken.check() is called at each phase boundary.
        """

    def _emit(
        self,
        session: RunSession,
        event_type: str,
        content: str,
        metadata: dict[str, Any] | None = None,
        agent_name: str | None = None,
    ) -> None:
        """Append a RunEvent to EventStore."""

    def _update_status(self, session: RunSession, status: str) -> RunSession:
        """Update session status in SessionStore and return updated session."""

    def _mark_failed(self, session: RunSession, error: str) -> None:
        """Mark session as failed, emit SYSTEM_END."""
```

---

### 2.5 `phases/phase0_workspace.py`

**Purpose:** Create the directory structure and materialize stable scaffold files.

**Dependencies:** `workspace/scaffold_service.py` (existing), `core/handoff_models.py`, `core/handoff_store.py`

```python
from __future__ import annotations

from pathlib import Path
from typing import Any

from core.handoff_models import WorkspaceSetupResult
from core.handoff_store import HandoffStore
from workspace.scaffold_service import ScaffoldService


class WorkspaceSetupService:
    """
    Phase 0: Set up the workspace directory structure.

    Creates:
        output_path/
        output_path/workspace/          ← code sandbox
        output_path/workspace/src/
        output_path/workspace/results/
        output_path/workspace/logs/
        output_path/handoff/
        output_path/paper/

    Then materializes stable scaffold files from scaffolds/{profile_name}/.

    Args:
        scaffold_service (ScaffoldService): Existing service (reused).
    """

    def __init__(self, scaffold_service: ScaffoldService) -> None: ...

    def setup(
        self,
        output_path: Path,
        research_input: dict[str, Any],
        profile_name: str,
        emit: Any,  # callable(event_type, content, metadata)
    ) -> WorkspaceSetupResult:
        """
        Set up directories and scaffold.

        Returns:
            WorkspaceSetupResult with workspace_root, scaffold_files, profile_name.

        Side effects:
            Writes handoff/workspace_setup.json via HandoffStore.
            Emits WORKSPACE_GENERATION_START and WORKSPACE_GENERATION_DONE events.
        """
```

---

### 2.6 `phases/phase1_planning.py`

**Purpose:** Run Planner + Designer Crew. Parse outputs into `PlanBundle`. Handle revision cycles.

**Dependencies:** `crewai`, `crew_agents/`, `core/handoff_models.py`, `core/json_extractor.py`

```python
from __future__ import annotations

from pathlib import Path
from typing import Any, Callable

from crewai import LLM, Crew, Task

from core.handoff_models import PlanBundle, PlannerResult, DesignerResult
from core.handoff_store import HandoffStore
from crew_agents.planner import make_planner_agent
from crew_agents.designer import make_designer_agent


class PlannerDesignerService:
    """
    Phase 1: Planning and Design.

    Runs a two-task Crew:
        Task 1: Planner — produces PlannerResult JSON
        Task 2: Designer — produces DesignerResult JSON (workspace_structure)

    Then synthesizes FileNodeSpec list and dep_graph from DesignerResult.
    Then partitions files into StagedFileList (stage 1/2/3).

    Constructor Args:
        llm (LLM): LLM for both agents (or use create_llm_for_agent per agent).
    """

    def __init__(self, llm: LLM) -> None: ...

    def run(
        self,
        research_input: dict[str, Any],
        workspace_root: Path,
        emit: Callable,
        user_feedback: str | None = None,
        revision_count: int = 0,
        previous_feedback_history: list[str] | None = None,
    ) -> PlanBundle:
        """
        Execute Phase 1 and return PlanBundle.

        Args:
            research_input: Normalized request dict.
            workspace_root: For HandoffStore.
            emit: Event emitter callable.
            user_feedback: Rejection feedback from previous approval round (injected into prompts).
            revision_count: How many times we've been through reject+revise.
            previous_feedback_history: All prior feedback for context.

        Returns:
            PlanBundle with planner_result, designer_result, file_tree, dep_graph, staged_files.

        Side effects:
            Writes handoff/plan_bundle.json.
            Emits AGENT_THINKING, AGENT_MESSAGE, PHASE_COMPLETE events.

        Internal steps:
            1. Build task descriptions (inject user_feedback if present)
            2. Build Crew with step/task callbacks
            3. crew.kickoff(inputs=...)
            4. Extract and validate PlannerResult from task 0 output
            5. Extract and validate DesignerResult from task 1 output
            6. Call _build_file_tree(designer_result) -> list[FileNodeSpec]
            7. Call _build_dep_graph(file_tree) -> dict[str, list[str]]
            8. Call _assign_stages(file_tree) -> StagedFileList
            9. Return PlanBundle(...)
        """

    def _build_file_tree(self, designer_result: DesignerResult) -> list:
        """
        Convert DesignerResult.workspace_structure.files (list[FileSpec])
        into list[FileNodeSpec] with stage assignments.

        Stage assignment heuristic:
            stage1: files with no .py extension, or files in root of workspace
                    (pyproject.toml, requirements.txt, config.json, etc.)
            stage2: .py files that are not the entry point
            stage3: src/main.py, src/experiment_impl.py, or any file listed last
                    in DesignerResult.workspace_structure.generation_order
        """

    def _build_dep_graph(self, file_tree: list) -> dict[str, list[str]]:
        """Build adjacency list from FileNodeSpec.imports_from fields."""

    def _assign_stages(self, file_tree: list) -> Any:
        """Group file paths by stage into StagedFileList."""

    def _build_planner_task(
        self, research_input: dict[str, Any], user_feedback: str | None
    ) -> Task:
        """
        Build the Planner Task.

        Task description includes:
            - Research topic, goal, domain, data description
            - If user_feedback: prepend "REVISION REQUEST: {user_feedback}\n\n"
            - Output format: PlannerResult JSON schema
        Expected output: PlannerResult JSON
        """

    def _build_designer_task(
        self, research_input: dict[str, Any], planner_task: Task
    ) -> Task:
        """
        Build the Designer Task (depends on planner_task).

        Task description includes:
            - Reference to planner output
            - Instruction to produce FileNodeSpec-compatible file specs
            - Instruction to include stage assignment for each file
        Expected output: DesignerResult JSON
        """
```

---

### 2.7 `phases/phase2_coding.py`

**Purpose:** Staged coding with per-file syntax/import checks, LLM repair, and user guidance escalation. Never gives up.

**Dependencies:** `crewai`, `crew_agents/file_coder.py`, `crew_tools/syntax_tools.py`, `orchestration/guidance_registry.py`, `core/handoff_models.py`

```python
from __future__ import annotations

import time
from pathlib import Path
from typing import Any, Callable, Optional

from crewai import LLM, Crew, Task

from core.handoff_models import (
    PlanBundle, CodingResult, StageResult, StageCheckResult,
    FileCheckError, RepairRecord, FileNodeSpec, StagedFileList,
)
from core.handoff_store import HandoffStore
from crew_agents.file_coder import make_file_coder_agent
from crew_tools.syntax_tools import SyntaxCheckTool, ImportCheckTool
from orchestration.cancellation_token import CancellationToken
from orchestration.guidance_registry import GuidanceRegistry


_STAGE_NAMES = {
    1: "scaffold_and_config",
    2: "utility_modules",
    3: "entry_point",
}


class StagedCoderService:
    """
    Phase 2: Staged coding loop.

    For each stage (1, 2, 3):
        1. Build one Task per file in the stage
        2. Run a Crew with FileCoder agent
        3. Run syntax/import check on all files in stage
        4. If check passes: emit STAGE_CHECK_PASS, continue to next stage
        5. If check fails: enter _repair_loop()

    _repair_loop(file, error):
        for attempt in range(1, max_llm_repair_attempts + 1):
            token.check()
            LLM-generate repair via single-task Crew
            re-run check
            if pass: emit FILE_FIXED, return
            emit REPAIR_ATTEMPT with result
        # LLM exhausted
        emit USER_GUIDANCE_NEEDED
        update status to "awaiting_guidance"
        guidance_state = guidance_registry.open(run_id, ...)
        resolved = guidance_state.wait(timeout=guidance_timeout)
        if not resolved:
            update status to "paused"
            pause_event.wait()  ← blocks until cancel() or guidance arrives later
        inject guidance into next repair prompt
        reset attempt counter
        goto top of repair loop  ← NEVER breaks out of loop without success or cancel

    Constructor Args:
        llm (LLM): FileCoder LLM.
        guidance_registry (GuidanceRegistry): Shared with API routes.
        max_llm_repair_attempts (int): LLM auto-repair cap before asking user (default 5).
        guidance_timeout_seconds (float): Seconds to wait for guidance before pausing (default 7200).
    """

    def __init__(
        self,
        llm: LLM,
        guidance_registry: GuidanceRegistry,
        max_llm_repair_attempts: int = 5,
        guidance_timeout_seconds: float = 7200.0,
    ) -> None: ...

    def run(
        self,
        plan_bundle: PlanBundle,
        workspace_root: Path,
        run_id: str,
        emit: Callable,
        update_status: Callable[[str], None],
        token: CancellationToken,
    ) -> CodingResult:
        """
        Execute all three coding stages in order.

        Args:
            plan_bundle: PlanBundle from Phase 1.
            workspace_root: Sandbox root.
            run_id: For GuidanceRegistry keying.
            emit: Event emitter callable.
            update_status: Callable(status_string) to update SessionStore.
            token: CancellationToken checked before each repair attempt.

        Returns:
            CodingResult with stage_results, all_files_passed=True.
            (Only returns True because the loop never exits on failure.)

        Side effects:
            Writes all workspace source files.
            Writes handoff/coding_result.json.
        """

    def _run_stage(
        self,
        stage: int,
        files: list[FileNodeSpec],
        plan_bundle: PlanBundle,
        workspace_root: Path,
        run_id: str,
        emit: Callable,
        update_status: Callable,
        token: CancellationToken,
    ) -> StageResult:
        """
        Code all files in one stage, then check + repair until passing.

        Steps:
            1. emit STAGE_START
            2. _run_coding_crew(stage, files, plan_bundle, workspace_root)
            3. check_result = _run_stage_check(stage, files, workspace_root)
            4. if check_result.passed: emit STAGE_CHECK_PASS, return StageResult
            5. for each error in check_result.errors:
                   _repair_loop(error.file_path, error, plan_bundle, workspace_root,
                                run_id, emit, update_status, token, repair_records)
            6. re-run _run_stage_check
            7. goto 4
        """

    def _run_coding_crew(
        self,
        stage: int,
        files: list[FileNodeSpec],
        plan_bundle: PlanBundle,
        workspace_root: Path,
    ) -> None:
        """
        Build one Task per file and run a Crew with FileCoder agent.

        Task description for each file includes:
            - file path and responsibility from FileNodeSpec
            - exports / imports_from from FileNodeSpec
            - coder_context: JSON blob with stack_rule, workspace_root, plan summary
            - "Verify by running: python -m py_compile {path}"
        """

    def _run_stage_check(
        self,
        stage: int,
        files: list[FileNodeSpec],
        workspace_root: Path,
    ) -> StageCheckResult:
        """
        Run SyntaxCheckTool (and ImportCheckTool for stage 2+) on all files.

        For stage 1: py_compile only (no import check — config files may not be importable)
        For stage 2+: py_compile + import check

        Returns StageCheckResult with passed=True if all files pass.
        """

    def _repair_loop(
        self,
        file_path: str,
        initial_error: FileCheckError,
        plan_bundle: PlanBundle,
        workspace_root: Path,
        run_id: str,
        emit: Callable,
        update_status: Callable,
        token: CancellationToken,
        repair_records: list[RepairRecord],
        pending_guidance: Optional[str] = None,
    ) -> None:
        """
        LLM repair loop for a single file. Blocks until the file passes checks.

        Algorithm:
            attempt = 0
            guidance = pending_guidance
            while True:
                token.check()  ← raises CancelledError if cancelled
                attempt += 1
                emit REPAIR_ATTEMPT(file, attempt, guidance_used=guidance)
                run single-task FileCoder Crew with:
                    - current file content (read via WorkspaceReadTool)
                    - error description
                    - guidance (if non-None, prepend to task description)
                check_result = _check_single_file(file_path, workspace_root)
                repair_records.append(RepairRecord(..., passed_after=check_result.passed))
                if check_result.passed:
                    emit FILE_FIXED
                    return
                emit STAGE_CHECK_FAIL
                if attempt < max_llm_repair_attempts:
                    continue
                # Ask user for guidance
                emit USER_GUIDANCE_NEEDED(file, error, attempt)
                update_status("awaiting_guidance")
                req = guidance_registry.open(run_id, file_path, ...)
                resolved = req.wait(guidance_timeout)
                if resolved:
                    guidance = req.guidance
                    update_status("running")
                    emit USER_GUIDANCE_RECEIVED(file, guidance)
                    attempt = 0
                    continue
                else:
                    # Timeout: pause the run
                    update_status("paused")
                    # Wait indefinitely until guidance arrives or cancel
                    req.wait(timeout=float("inf"))   ← CancelledError will interrupt
                    if req.is_delivered:
                        guidance = req.guidance
                        update_status("running")
                        emit USER_GUIDANCE_RECEIVED(...)
                        attempt = 0
                        continue
                    # cancel() was called — CancellationToken.check() will raise on next iteration

        Never returns without the file passing (only exits via CancelledError).
        """

    def _check_single_file(
        self, file_path: str, workspace_root: Path
    ) -> StageCheckResult:
        """Run syntax + import check on one file. Returns StageCheckResult."""

    def _build_repair_task(
        self,
        file_path: str,
        error: FileCheckError,
        workspace_root: Path,
        plan_bundle: PlanBundle,
        guidance: Optional[str],
    ) -> Task:
        """
        Build a single-task description for repair.

        Task description:
            - "You are fixing a broken file: {file_path}"
            - "Current error: {error.error_type}: {error.error_message} at line {error.line_number}"
            - If guidance: "USER GUIDANCE: {guidance}\n\nApply this guidance when rewriting the file."
            - "Read the current file with WorkspaceReadTool. Rewrite it. Verify with py_compile."
        """
```

---

### 2.8 `phases/phase3_execution.py`

**Purpose:** Run the experiment subprocess, collect results, emit structured events.

**Dependencies:** `crew_agents/executor.py`, `core/handoff_models.py`, `crew_tools/execution_tools.py`

```python
from __future__ import annotations

from pathlib import Path
from typing import Any, Callable

from crewai import LLM

from core.handoff_models import CodingResult, ExecutionResult
from core.handoff_store import HandoffStore
from crew_agents.executor import make_executor_agent
from orchestration.cancellation_token import CancellationToken


class ExecutionService:
    """
    Phase 3: Run the experiment.

    Builds a single-task Crew with the ExecutorAgent.
    The Executor calls RunCommandTool then ReadResultTool.
    Results are parsed from workspace/results/result.json.

    Constructor Args:
        llm (LLM): Executor LLM.
        subprocess_timeout (int): Passed to RunCommandTool.
    """

    def __init__(self, llm: LLM, subprocess_timeout: int = 3600) -> None: ...

    def run(
        self,
        coding_result: CodingResult,
        workspace_root: Path,
        emit: Callable,
        token: CancellationToken,
    ) -> ExecutionResult:
        """
        Execute Phase 3.

        Steps:
            1. token.check()
            2. emit EXECUTION_START
            3. Build Crew with single ExecutorTask
            4. crew.kickoff(inputs={"workspace_root": str(workspace_root), ...})
            5. Parse executor output to extract return_code, duration_s, metrics
            6. If result.json exists, load it to populate metrics dict
            7. emit EXECUTION_DONE
            8. Write handoff/execution_result.json
            9. return ExecutionResult

        Note: This phase does NOT retry on failure. The Executor agent's
              task description asks it to report failures accurately.
              Re-running experiments on failure is out of scope for V4
              (the research plan specifies the experiment; if it fails,
              the Writer will note the failure in the paper).
        """
```

---

### 2.9 `phases/phase4_writing.py`

**Purpose:** Generate paper section by section. Self-verify each section before proceeding.

**Dependencies:** `crew_agents/writer.py`, `crew_tools/report_tools.py`, `core/handoff_models.py`

```python
from __future__ import annotations

from pathlib import Path
from typing import Any, Callable

from crewai import LLM

from core.handoff_models import PlanBundle, ExecutionResult, WritingResult, SectionDraft
from core.handoff_store import HandoffStore
from crew_agents.writer import make_writer_agent
from orchestration.cancellation_token import CancellationToken


_SECTION_ORDER = [
    "Introduction",
    "Related Works",
    "Proposed Method",
    "Experiments",
    "Conclusion",
    "References",
    "Abstract",
]


class WriterService:
    """
    Phase 4: Section-by-section paper generation.

    For each section in _SECTION_ORDER:
        1. Build a single-task Crew for that section
        2. Run the Crew
        3. Parse output: section content + self_verify_notes
        4. If self_verify_passed=False: re-run once
        5. Emit SECTION_DRAFT_DONE
        6. Write paper/{section_name}.md via WriteReportSectionTool

    After all sections:
        Concatenate into paper/paper.md in canonical order.
        Emit PAPER_DONE.

    Constructor Args:
        llm (LLM): Writer LLM.
    """

    def __init__(self, llm: LLM) -> None: ...

    def run(
        self,
        plan_bundle: PlanBundle,
        execution_result: ExecutionResult,
        workspace_root: Path,
        emit: Callable,
        token: CancellationToken,
    ) -> WritingResult:
        """
        Execute Phase 4.

        Returns WritingResult with paper_path, sections, total_word_count.
        Writes handoff/writing_result.json.
        """

    def _run_section(
        self,
        section_name: str,
        plan_bundle: PlanBundle,
        execution_result: ExecutionResult,
        workspace_root: Path,
        completed_sections: list[SectionDraft],
    ) -> SectionDraft:
        """
        Write one section.

        Task description includes:
            - Section name and role ("You are writing the {section_name} section")
            - Context: planner_result, designer_result, metrics, stdout_tail
            - Previously completed sections (titles + first 200 chars each) for coherence
            - "Self-verify: after writing, list what you checked (citations, numbers, logic)"
            - "Use WriteReportSectionTool to save the section"

        Returns SectionDraft with content, word_count, self_verify_passed, self_verify_notes.
        """

    def _assemble_paper(self, sections: list[SectionDraft], paper_dir: Path) -> Path:
        """
        Concatenate section files into paper/paper.md.

        Order: Introduction → Related Works → Proposed Method →
               Experiments → Conclusion → References → Abstract

        Returns path to assembled paper.md.
        """
```

---

### 2.10 `api/routes/approval.py`

**Purpose:** Deliver plan approval/rejection and user guidance from HTTP requests to waiting pipeline threads.

```python
from __future__ import annotations

from fastapi import APIRouter, HTTPException, Request

from api.schemas import ApproveRequest, ApproveResponse, GuidanceRequest, GuidanceResponse

router = APIRouter(prefix="/api/v1/runs", tags=["approval"])


@router.post("/{run_id}/approve")
async def approve_plan(
    run_id: str,
    payload: ApproveRequest,
    request: Request,
) -> ApproveResponse:
    """
    Deliver plan approval or rejection.

    1. Verify run is in status=awaiting_approval (409 otherwise)
    2. Call services.orchestrator.approve_plan(run_id, payload.action, payload.feedback)
    3. Return ApproveResponse

    Raises:
        404: Unknown run_id
        409: Run not in awaiting_approval state
        422: action=reject with no feedback
    """


@router.post("/{run_id}/guidance")
async def deliver_guidance(
    run_id: str,
    payload: GuidanceRequest,
    request: Request,
) -> GuidanceResponse:
    """
    Inject user guidance into a stuck repair loop.

    1. Verify run is in status=awaiting_guidance or status=paused (409 otherwise)
    2. Call services.orchestrator.deliver_guidance(run_id, payload.guidance)
    3. Return GuidanceResponse

    Raises:
        404: Unknown run_id
        409: Run not awaiting guidance
    """
```

---

### 2.11 `crew_tools/syntax_tools.py`

**Purpose:** Provide syntax and import checking as CrewAI tools.

```python
from __future__ import annotations

import json
import py_compile
import subprocess
import sys
import tempfile
from pathlib import Path
from typing import Any, Type

from crewai.tools import BaseTool
from pydantic import BaseModel, Field


class _SyntaxCheckInput(BaseModel):
    workspace_root: str = Field(description="Absolute path to workspace root")
    relative_paths: list[str] = Field(
        description="List of relative .py file paths to syntax-check"
    )


class SyntaxCheckTool(BaseTool):
    """
    Run py_compile on a list of Python files in the workspace.

    Returns JSON:
    {
      "passed": bool,
      "results": [
        { "path": str, "ok": bool, "error": str | null, "line": int | null }
      ]
    }

    Implementation:
        For each path:
            try:
                py_compile.compile(str(abs_path), doraise=True)
                results.append({"path": rel, "ok": True, "error": None, "line": None})
            except py_compile.PyCompileError as exc:
                # exc.msg contains the error; exc.lineno contains the line
                results.append({"path": rel, "ok": False, "error": str(exc.msg), "line": exc.lineno})
        passed = all(r["ok"] for r in results)
    """

    name: str = "SyntaxCheckTool"
    description: str = (
        "Check Python files for syntax errors using py_compile. "
        "Returns JSON with 'passed' bool and per-file results."
    )
    args_schema: Type[BaseModel] = _SyntaxCheckInput

    def _run(self, workspace_root: str, relative_paths: list[str]) -> str: ...


class _ImportCheckInput(BaseModel):
    workspace_root: str = Field(description="Absolute path to workspace root")
    module_path: str = Field(
        description="Relative path of the Python file to import-check (e.g. 'src/utils.py')"
    )


class ImportCheckTool(BaseTool):
    """
    Attempt to import a Python module by spawning an isolated subprocess.

    The subprocess runs:
        import sys
        sys.path.insert(0, workspace_root)
        import importlib.util
        spec = importlib.util.spec_from_file_location("_mod_", abs_path)
        mod = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(mod)

    Returns JSON:
    { "passed": bool, "error": str | null }

    Implementation:
        Writes the above snippet to a temp file.
        subprocess.run(["python", temp_script], capture_output=True, timeout=30,
                       env={...PYTHONPATH: workspace_root + os.pathsep + existing})
        passed = (returncode == 0)
        error = stderr if not passed else None
    """

    name: str = "ImportCheckTool"
    description: str = (
        "Import a Python module in a clean subprocess to detect ImportError or ModuleNotFoundError. "
        "Returns JSON with 'passed' bool and 'error' string."
    )
    args_schema: Type[BaseModel] = _ImportCheckInput

    def _run(self, workspace_root: str, module_path: str) -> str: ...
```

---

## 3. Import Graph

```
api/app.py
    → api/routes/runs.py
        → api/schemas.py
        → orchestration/pipeline_orchestrator.py
    → api/routes/approval.py
        → api/schemas.py
        → orchestration/pipeline_orchestrator.py
            → orchestration/approval_gate.py
            → orchestration/guidance_registry.py
            → orchestration/cancellation_token.py
            → phases/phase0_workspace.py
                → workspace/scaffold_service.py
                → core/handoff_models.py
                → core/handoff_store.py
            → phases/phase1_planning.py
                → crewai
                → crew_agents/planner.py
                → crew_agents/designer.py
                → core/handoff_models.py
                → core/json_extractor.py
            → phases/phase2_coding.py
                → crewai
                → crew_agents/file_coder.py
                    → crew_tools/workspace_tools.py
                    → crew_tools/execution_tools.py
                    → crew_tools/edit_tool.py
                    → crew_tools/syntax_tools.py   ← NEW
                → crew_tools/syntax_tools.py       ← NEW (direct check)
                → orchestration/guidance_registry.py
                → core/handoff_models.py
            → phases/phase3_execution.py
                → crewai
                → crew_agents/executor.py
                    → crew_tools/execution_tools.py
                → core/handoff_models.py
            → phases/phase4_writing.py
                → crewai
                → crew_agents/writer.py
                    → crew_tools/workspace_tools.py
                    → crew_tools/report_tools.py
                → core/handoff_models.py
            → runtime/event_store.py
            → runtime/session_store.py
            → runtime/models.py
            → core/llm_factory.py
            → core/llm_retry.py
    → api/routes/stream.py
        → runtime/stream_service.py
            → runtime/event_store.py
            → runtime/session_store.py
            → runtime/state_calculator.py
    → api/routes/artifacts.py
        → runtime/run_repository.py
```

---

## 4. Key Invariants for Implementers

1. **The repair loop NEVER exits on failure.** `StagedCoderService._repair_loop()` only returns when `_check_single_file()` passes. The only other exit is `CancelledError` raised by `CancellationToken.check()`.

2. **Approval gate blocks the pipeline thread, not the web server.** `ApprovalGateState.wait()` calls `threading.Event.wait()` inside the daemon thread. FastAPI/uvicorn's event loop is never blocked.

3. **SSE stream continues during all waits.** `StreamService.stream_logs()` is an async generator that polls `EventStore.tail()` every second. It runs in the asyncio event loop regardless of whether the pipeline thread is blocked on an approval gate or guidance request.

4. **All file writes go through `WorkspaceWriteTool`.** Direct `Path.write_text()` is only used by `WorkspaceSetupService` for scaffold files and by `WriterService._assemble_paper()`. All agent-generated code goes through the tool, which enforces the path-traversal sandbox.

5. **`HandoffStore` is the single source of truth between phases.** Each phase reads its inputs from the previous phase's handoff JSON and writes its own. This makes each phase independently resumable in future versions.

6. **`LLMCostTracker` is reset per run.** `PipelineOrchestrator._execute()` calls `get_cost_tracker().reset()` at the start. Cost is emitted in `SYSTEM_END` metadata.

7. **`CancellationToken.check()` is called at minimum:** at phase transitions (0→1, 1→2, 2→3, 3→4), at the top of every repair loop iteration, and before each LLM Crew kickoff.

8. **Guidance timeout goes to "paused", not "failed".** A paused run can be resumed by `POST /runs/{id}/guidance`. Only `DELETE /runs/{id}` transitions a paused run to "cancelled/failed". This preserves all work done so far.
