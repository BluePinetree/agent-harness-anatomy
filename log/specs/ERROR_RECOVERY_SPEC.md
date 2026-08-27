# ERROR_RECOVERY_SPEC.md
# Error Taxonomy, Recovery Strategies, and Escalation Protocol

> Version: 1.0  
> Last updated: 2026-05-19  
> Status: Authoritative specification for implementation

---

## Table of Contents

1. [Error Taxonomy](#1-error-taxonomy)
2. [Detection Patterns (Regex + Logic)](#2-detection-patterns-regex--logic)
3. [Recovery Strategies by Error Type](#3-recovery-strategies-by-error-type)
4. [Escalation Thresholds and Triggers](#4-escalation-thresholds-and-triggers)
5. [User Interaction Message Formats](#5-user-interaction-message-formats)
6. [State Persistence During Error Recovery](#6-state-persistence-during-error-recovery)
7. [The "Fixable vs Human Judgment" Decision Tree](#7-the-fixable-vs-human-judgment-decision-tree)
8. [Error Classification Implementation](#8-error-classification-implementation)

---

## 1. Error Taxonomy

Errors are organized in a five-level hierarchy. The level determines who handles the error (auto-repair, LLM-guided repair, or human) and how urgently escalation occurs.

```
Level 1 — SYNTAX ERROR
  Definition : The file cannot be parsed as valid Python (or target language).
  Source     : SyntaxChecker (python -c "import ast; ast.parse(...)")
  Fixability : High — LLM can reliably fix these with error message + line number.
  Auto-repair budget: MAX_AUTO_ATTEMPTS (default: 5)
  Examples   :
    - SyntaxError: invalid syntax
    - IndentationError: unexpected indent
    - TabError: inconsistent use of tabs and spaces

Level 2 — IMPORT ERROR
  Definition : The file parses but cannot be imported — missing module, name error
               at module level, or circular import.
  Source     : ImportChecker
  Fixability : Medium — dependency issues need context about the full file tree.
  Auto-repair budget: MAX_AUTO_ATTEMPTS (default: 5)
  Sub-types  :
    2a. ModuleNotFoundError  — missing third-party package (pip install may help)
    2b. ImportError          — name not exported from module
    2c. CircularImportError  — two modules import each other at module level
    2d. AttributeError       — attribute does not exist on imported object
    2e. DLL/OS Error         — binary dependency issue (IMPORT_SKIP; not code error)

Level 3 — RUNTIME ERROR
  Definition : The code runs but raises an exception during execution.
  Source     : ExecutorAgent (captures stderr)
  Fixability : Variable — depends on whether the error is in generated code or
               in third-party code called by generated code.
  Auto-repair budget: MAX_EXEC_ATTEMPTS (default: 3)
  Sub-types  :
    3a. ValueError / TypeError   — data shape or type mismatch
    3b. KeyError / IndexError    — wrong key/index into data structure
    3c. FileNotFoundError        — expected file doesn't exist (pipeline artifact missing)
    3d. MemoryError              — model/data too large for available RAM
    3e. TimeoutError             — experiment exceeded EXEC_TIMEOUT_SECONDS
    3f. CUDA / GPU Error         — GPU-specific failures

Level 4 — RESULT ERROR
  Definition : The experiment runs to completion but result.json is missing,
               malformed, or contains anomalous values.
  Source     : ExecutorAgent (ReadResultTool + validation)
  Fixability : Low — usually indicates logic error in result serialization.
  Auto-repair budget: MAX_RESULT_REPAIR_ATTEMPTS (default: 2)
  Sub-types  :
    4a. MissingResult         — result.json not written
    4b. MalformedResult       — result.json unparseable
    4c. EmptyMetrics          — metrics dict is empty or all None
    4d. AnomalousMetrics      — values outside plausible range (e.g., accuracy = 0.0 or > 1.0)

Level 5 — LLM ERROR
  Definition : The LLM itself fails (API error, timeout, malformed response).
  Source     : Any agent that calls the LLM API
  Fixability : High for transient errors; low for persistent content issues.
  Auto-repair budget: MAX_LLM_RETRIES (default: 10 with exponential backoff)
  Sub-types  :
    5a. RateLimitError (429)   — rate limit; retry with backoff
    5b. OverloadError (529)    — server overload; retry with backoff
    5c. APITimeoutError        — network timeout; retry with backoff
    5d. InvalidResponseError   — LLM output cannot be parsed as expected JSON
    5e. EmptyResponseError     — LLM returned empty content block
    5f. RefusalError           — LLM refused to generate (safety/content filter)
```

---

## 2. Detection Patterns (Regex + Logic)

### Level 1: Syntax Errors

```python
import ast, re

class SyntaxChecker:

    def check(self, file_path: str) -> CheckResult:
        try:
            source = open(file_path, encoding="utf-8").read()
            ast.parse(source)
            return CheckResult(ok=True)
        except SyntaxError as e:
            return CheckResult(
                ok=False,
                error_type=ErrorType.SYNTAX,
                error_class="SyntaxError",
                message=str(e),
                line_number=e.lineno,
                col_offset=e.offset,
                file_path=file_path,
            )
        except IndentationError as e:
            return CheckResult(
                ok=False,
                error_type=ErrorType.SYNTAX,
                error_class="IndentationError",
                message=str(e),
                line_number=e.lineno,
                file_path=file_path,
            )
```

### Level 2: Import Errors

```python
import importlib, subprocess, sys

# DLL/OS errors are NOT code errors — skip them
IMPORT_SKIP_PATTERNS = [
    r"DLL load failed",
    r"cannot open shared object",
    r"The specified module could not be found",
    r"library not found",
]

class ImportChecker:

    def check(self, file_path: str, workspace_root: str) -> CheckResult:
        # derive module name from path
        rel = Path(file_path).relative_to(workspace_root)
        module = str(rel).replace("/", ".").replace("\\", ".").removesuffix(".py")

        result = subprocess.run(
            [sys.executable, "-c", f"import sys; sys.path.insert(0, '{workspace_root}'); import {module}"],
            capture_output=True, text=True, timeout=30
        )

        if result.returncode == 0:
            return CheckResult(ok=True)

        stderr = result.stderr

        # Check if it's a DLL/OS error (not our code's fault)
        for pattern in IMPORT_SKIP_PATTERNS:
            if re.search(pattern, stderr, re.IGNORECASE):
                return CheckResult(ok=True, warning="IMPORT_SKIP: DLL/OS error, not a code issue")

        # Classify import error
        error_class = "ImportError"
        if "ModuleNotFoundError" in stderr:
            error_class = "ModuleNotFoundError"
            missing_module = re.search(r"No module named '([^']+)'", stderr)
            missing_name = missing_module.group(1) if missing_module else "unknown"
        elif "circular" in stderr.lower() or "partially initialized" in stderr.lower():
            error_class = "CircularImportError"
            missing_name = None
        elif "cannot import name" in stderr:
            error_class = "ImportError"
            missing_name = re.search(r"cannot import name '([^']+)'", stderr)
            missing_name = missing_name.group(1) if missing_name else "unknown"
        else:
            missing_name = None

        return CheckResult(
            ok=False,
            error_type=ErrorType.IMPORT,
            error_class=error_class,
            message=stderr[-2000:],   # last 2000 chars
            file_path=file_path,
            extra={"missing_name": missing_name},
        )
```

### Level 3: Runtime Errors — Stderr Pattern Matching

```python
RUNTIME_ERROR_PATTERNS = {
    "ValueError":           re.compile(r"ValueError:\s*(.+)"),
    "TypeError":            re.compile(r"TypeError:\s*(.+)"),
    "KeyError":             re.compile(r"KeyError:\s*(.+)"),
    "IndexError":           re.compile(r"IndexError:\s*(.+)"),
    "FileNotFoundError":    re.compile(r"FileNotFoundError:\s*(.+)"),
    "MemoryError":          re.compile(r"MemoryError"),
    "CUDA_OOM":             re.compile(r"CUDA out of memory|RuntimeError:.*CUDA"),
    "TimeoutKilled":        re.compile(r"Killed|TimeoutExpired|signal 9"),
    "NameError":            re.compile(r"NameError:\s*(.+)"),
    "AttributeError":       re.compile(r"AttributeError:\s*(.+)"),
    "ZeroDivisionError":    re.compile(r"ZeroDivisionError"),
    "RecursionError":       re.compile(r"RecursionError|maximum recursion depth"),
}

def classify_runtime_error(stderr: str) -> RuntimeErrorInfo:
    for name, pattern in RUNTIME_ERROR_PATTERNS.items():
        match = pattern.search(stderr)
        if match:
            detail = match.group(1) if match.lastindex else ""
            # extract traceback: last 20 non-empty lines of stderr
            tb_lines = [l for l in stderr.split("\n") if l.strip()][-20:]
            return RuntimeErrorInfo(
                error_class=name,
                message=detail,
                traceback="\n".join(tb_lines),
                raw_stderr=stderr[-2000:],
            )
    return RuntimeErrorInfo(
        error_class="UnknownRuntimeError",
        message="",
        traceback=stderr[-2000:],
        raw_stderr=stderr[-2000:],
    )
```

### Level 4: Result Errors

```python
import json

PLAUSIBLE_METRIC_RANGES = {
    "accuracy":     (0.0, 1.0),
    "f1":           (0.0, 1.0),
    "precision":    (0.0, 1.0),
    "recall":       (0.0, 1.0),
    "auc":          (0.0, 1.0),
    "loss":         (0.0, 1e6),
    "mse":          (0.0, 1e9),
    "mae":          (0.0, 1e9),
    "r2":           (-1e6, 1.0),
}

def validate_result_json(result_path: str) -> CheckResult:
    if not Path(result_path).exists():
        return CheckResult(ok=False, error_type=ErrorType.RESULT,
                           error_class="MissingResult",
                           message=f"result.json not found at {result_path}")

    try:
        data = json.loads(Path(result_path).read_text())
    except json.JSONDecodeError as e:
        return CheckResult(ok=False, error_type=ErrorType.RESULT,
                           error_class="MalformedResult",
                           message=str(e))

    metrics = data.get("metrics", {})
    if not metrics:
        return CheckResult(ok=False, error_type=ErrorType.RESULT,
                           error_class="EmptyMetrics",
                           message="metrics dict is empty or missing")

    for key, value in metrics.items():
        if isinstance(value, (int, float)):
            low_key = key.lower()
            for metric_name, (lo, hi) in PLAUSIBLE_METRIC_RANGES.items():
                if metric_name in low_key:
                    if not (lo <= value <= hi):
                        return CheckResult(
                            ok=False, error_type=ErrorType.RESULT,
                            error_class="AnomalousMetrics",
                            message=f"Metric '{key}'={value} outside plausible range [{lo}, {hi}]"
                        )

    return CheckResult(ok=True)
```

### Level 5: LLM Errors

```python
import anthropic

LLM_TRANSIENT_ERRORS = (
    anthropic.RateLimitError,
    anthropic.APITimeoutError,
    # OverloadError maps to anthropic.APIStatusError with status 529
)

def classify_llm_error(exc: Exception) -> LLMErrorInfo:
    if isinstance(exc, anthropic.RateLimitError):
        return LLMErrorInfo(error_class="RateLimitError", transient=True,
                            retry_after=int(exc.response.headers.get("retry-after", 60)))

    if isinstance(exc, anthropic.APIStatusError) and exc.status_code == 529:
        return LLMErrorInfo(error_class="OverloadError", transient=True, retry_after=30)

    if isinstance(exc, anthropic.APITimeoutError):
        return LLMErrorInfo(error_class="TimeoutError", transient=True, retry_after=10)

    if isinstance(exc, anthropic.BadRequestError):
        return LLMErrorInfo(error_class="InvalidRequestError", transient=False)

    return LLMErrorInfo(error_class="UnknownLLMError", transient=False)
```

---

## 3. Recovery Strategies by Error Type

### Level 1: Syntax Error Recovery

```
Strategy: Targeted patch with line-number context.

Agent prompt addition (injected into CoderAgent repair context):
  "File {file_path} has a SyntaxError at line {line_number}:
   {error_message}
   The code at that line is: {line_content}
   Fix ONLY the syntax error. Do not rewrite the entire file.
   Use workspace_write tool to overwrite the file."

Auto-repair budget: MAX_AUTO_ATTEMPTS attempts.
If still failing after budget:
  - Error is likely structural (LLM keeps producing same mistake)
  - Escalate to user with full fix_history
```

### Level 2a: ModuleNotFoundError Recovery

```
Strategy: First attempt pip install; if that fails, rewrite import.

Step 1 (attempt 1):
  Run: pip install {missing_module}
  Re-check import.
  If ok → done.

Step 2 (attempts 2–MAX_AUTO_ATTEMPTS):
  CoderAgent prompt:
    "ModuleNotFoundError: No module named '{missing_module}'
     Options in priority order:
     1. If this is a standard library module spelled wrong, fix the spelling.
     2. If this is a known equivalent (e.g., sklearn → scikit-learn), use the correct import.
     3. If this is an internal module, ensure the module exists in the workspace.
     4. If optional, guard with try/except ImportError.
     Do NOT add pip install calls inside source files."
```

### Level 2b: ImportError (name not found) Recovery

```
Strategy: Inspect the source module to find correct name.

CoderAgent repair prompt:
  "ImportError: cannot import name '{missing_name}' from '{module_path}'.
   The actual names exported by that module are: {actual_exports}
   (obtained by reading the module's __all__ or top-level definitions)
   Fix the import statement to use one of the available names,
   or add the missing name to {module_path} if it should exist there."
```

### Level 2c: Circular Import Recovery

```
Strategy: Move import inside function, or refactor to avoid cycle.

CoderAgent repair prompt:
  "Circular import detected: {module_a} imports {module_b} which imports {module_a}.
   Choose ONE of:
   1. Move the import of {module_b} inside the function that uses it (lazy import).
   2. Create a new module {module_c} that both {module_a} and {module_b} import from.
      Do NOT choose option 2 unless you also create {module_c} in the same repair.
   The import graph for context: {import_graph_excerpt}"
```

### Level 3: Runtime Error Recovery

```
Strategy selection based on error class:

ValueError/TypeError:
  Inject full traceback + the relevant data shapes.
  Prompt: "RuntimeError: {error_class}: {message}
           Traceback:
           {traceback}
           Fix the data shape mismatch. Do not change the algorithm logic."

MemoryError:
  Prompt: "MemoryError during execution.
           Reduce memory usage by:
           1. Use smaller batch sizes (default: 32 → 8)
           2. Use float32 instead of float64
           3. Process data in chunks
           Modify only the config or the data loading code."

TimeoutError:
  Prompt: "Experiment exceeded {EXEC_TIMEOUT_SECONDS}s timeout.
           Reduce computational load:
           1. Reduce dataset size to {REDUCED_DATASET_SIZE} samples
           2. Reduce model complexity (fewer layers/parameters)
           3. Reduce number of epochs/iterations
           Preserve the algorithm structure."

CUDA_OOM:
  Prompt: "CUDA out of memory.
           1. Reduce batch_size in config (try //= 2)
           2. Add torch.cuda.empty_cache() before model creation
           3. If still failing, fall back to CPU: device='cpu'"

FileNotFoundError:
  Check whether the missing file should have been created by an earlier stage.
  If yes → escalate immediately (pipeline artifact missing → systemic issue).
  If no → CoderAgent adds file creation to the relevant module.
```

### Level 4: Result Error Recovery

```
MissingResult:
  CoderAgent repair:
    "result.json was not written after execution.
     The main script must write results/result.json before exiting.
     Required format: { 'metrics': {...}, 'config': {...}, 'timestamp': '...' }
     Add the following at the end of {entry_point}:
       import json, pathlib
       pathlib.Path('results/result.json').write_text(json.dumps(results_dict, indent=2))"

MalformedResult:
  CoderAgent repair:
    "results/result.json is not valid JSON: {parse_error}
     Fix the serialization code in {entry_point}.
     Use json.dumps() with default=str to handle non-serializable types."

AnomalousMetrics:
  CoderAgent repair:
    "Metric '{metric_name}'={value} is outside the plausible range [{lo}, {hi}].
     This usually means:
     1. Division by zero in metric calculation → add epsilon
     2. Wrong normalization (multiplied by 100 when should be fraction)
     3. Metric computed on empty data → add assertion before metric calculation"
```

### Level 5: LLM Error Recovery

```python
import time, random

def with_llm_retry(fn, *args, max_retries=10, **kwargs):
    """
    Exponential backoff with jitter for transient LLM errors.
    Non-transient errors raise immediately.
    """
    BASE_DELAY_SEC  = 0.5
    MAX_DELAY_SEC   = 32.0
    JITTER_FRACTION = 0.25

    for attempt in range(max_retries):
        try:
            return fn(*args, **kwargs)
        except Exception as exc:
            info = classify_llm_error(exc)

            if not info.transient:
                raise   # non-transient: bubble up immediately

            if attempt == max_retries - 1:
                raise   # exhausted retries

            # compute backoff
            base = min(BASE_DELAY_SEC * (2 ** attempt), MAX_DELAY_SEC)
            jitter = random.random() * JITTER_FRACTION * base
            delay = base + jitter

            # use retry-after header if provided
            if info.retry_after:
                delay = max(delay, info.retry_after)

            time.sleep(delay)

    raise RuntimeError("LLM retry exhausted")  # unreachable but linters happy
```

---

## 4. Escalation Thresholds and Triggers

### Thresholds (all configurable via CONSTANTS.md)

| Error Level | Auto-Repair Budget | Escalation Trigger |
|-------------|--------------------|--------------------|
| Level 1 Syntax       | MAX_AUTO_ATTEMPTS (5)     | attempt_count > MAX_AUTO_ATTEMPTS |
| Level 2 Import       | MAX_AUTO_ATTEMPTS (5)     | attempt_count > MAX_AUTO_ATTEMPTS |
| Level 3 Runtime      | MAX_EXEC_ATTEMPTS (3)     | attempt_count > MAX_EXEC_ATTEMPTS |
| Level 4 Result       | MAX_RESULT_REPAIR (2)     | attempt_count > MAX_RESULT_REPAIR |
| Level 5 LLM transient| MAX_LLM_RETRIES (10)      | attempt > MAX_LLM_RETRIES |
| Level 5 LLM non-trans| 0 (immediate escalate)    | on first occurrence |

### "Immediate Escalation" Triggers (bypass auto-repair budget)

These conditions cause immediate escalation regardless of attempt count:

1. `FileNotFoundError` for a file that should have been created by an earlier pipeline stage
2. `CircularImportError` that persists after 2 auto-repair attempts (structural problem)
3. Any error where `fix_history` shows the same error message repeated 3+ times (LLM stuck in loop)
4. `CUDA OOM` with batch_size already at 1 (cannot reduce further)
5. `MemoryError` that persists after reducing dataset size to minimum
6. LLM `RefusalError` (content filter)

### Loop-Stuck Detection

```python
def is_repair_stuck(fix_history: list[dict]) -> bool:
    """
    Returns True if the last 3 auto-repair attempts produced the same error.
    This means the LLM is in a repair loop and cannot make progress.
    """
    if len(fix_history) < 3:
        return False
    last_three = fix_history[-3:]
    errors = [e.get("error", "") for e in last_three]
    # strip line numbers before comparing (line numbers may shift)
    normalized = [re.sub(r"line \d+", "line X", e) for e in errors]
    return len(set(normalized)) == 1   # all three are the same error
```

When `is_repair_stuck()` returns True, escalate immediately regardless of remaining budget.

---

## 5. User Interaction Message Formats

### Escalation Message: File Repair

```
============================================================
ACTION REQUIRED — Repair Escalation
============================================================
Run ID  : {run_id}
Phase   : {phase} (Stage {stage})
File    : {file_path}
------------------------------------------------------------
ERROR ({error_class}):
  {error_message}
  Line: {line_number}

AUTO-REPAIR HISTORY ({attempt_count} attempts):
  {attempt_1_description} → {attempt_1_result}
  {attempt_2_description} → {attempt_2_result}
  ...

------------------------------------------------------------
OPTIONS:
  [C] Continue  — grant {DEFAULT_EXTRA_ATTEMPTS} more auto-repair attempts
  [S] Skip      — write a minimal stub for this file and continue
  [H] Hint      — provide a fix hint to the coder agent
  [E] Edit      — edit the file yourself (type DONE when finished)

Stub behavior: the file will contain a valid but empty implementation.
  Functions will return None, classes will have pass bodies.
  The pipeline will continue; stubbed files are listed in the final report.
------------------------------------------------------------
Enter choice (C/S/H/E): _
```

### Escalation Message: Execution Failure

```
============================================================
ACTION REQUIRED — Experiment Execution Failed
============================================================
Run ID       : {run_id}
Entry point  : {entry_point}
Duration     : {duration_sec:.1f}s
Return code  : {return_code}
------------------------------------------------------------
STDERR (last 50 lines):
{stderr_tail}

------------------------------------------------------------
ATTEMPTED FIXES ({attempt_count} attempts):
  {fix_history_summary}

------------------------------------------------------------
OPTIONS:
  [C] Continue  — grant {DEFAULT_EXTRA_ATTEMPTS} more repair attempts
  [S] Skip      — record experiment as FAILED and continue to paper writing
                  (paper will note that results are unavailable)
  [H] Hint      — provide a diagnostic hint
  [E] Edit      — edit source files manually (type DONE when finished)
------------------------------------------------------------
Enter choice (C/S/H/E): _
```

### Escalation Message: Result Anomaly

```
============================================================
NOTICE — Anomalous Results Detected
============================================================
Run ID       : {run_id}
Result file  : {result_path}
------------------------------------------------------------
ISSUE:
  {anomaly_description}

ACTUAL RESULT FILE CONTENTS:
{result_json_preview}

------------------------------------------------------------
OPTIONS:
  [A] Accept    — accept results as-is and continue to writing
  [C] Continue  — attempt auto-repair of result serialization code
  [S] Skip      — treat results as unavailable
  [H] Hint      — provide correction hint
------------------------------------------------------------
Enter choice (A/C/S/H): _
```

### Approval Gate Message (Phase 1)

Detailed format specified in PIPELINE_SPEC.md §9. Key points:
- Shows full PlannerResult and DesignerResult in human-readable form
- APPROVE / REJECT / MODIFY options
- On MODIFY: user can specify individual fields to change:

```
MODIFY mode:
  Which fields to change?
  (press Enter to keep current value, or type new value)

  Problem Statement [current: ...]:
  Research Questions [current: ...]:
  Hypotheses [current: ...]:
  Success Criteria [current: ...]:
  Baseline Comparisons [current: ...]:
  Risk Items [current: ...]:
  
  Add to Designer constraints (leave blank to skip):
```

---

## 6. State Persistence During Error Recovery

### What to Save Before Any Blocking Operation

```python
@dataclass
class RepairCheckpoint:
    # Identity
    run_id:         str
    phase:          str              # "coding_stage_2", "execution", etc.
    file_path:      str | None       # which file is being repaired (None for execution)

    # Repair state
    attempt_count:  int
    max_auto:       int              # may have been increased by user
    fix_history:    list[dict]       # each: {attempt, action, outcome, error}
    stubbed:        bool

    # Context for resuming
    error_type:     str              # ErrorType enum value
    last_error:     str              # last error message
    last_error_class: str            # last error class name
    user_hint:      str | None       # hint provided by user, if any

    # Timestamps
    started_at:     float
    last_saved_at:  float

    # Pipeline context needed for repair
    workspace_root: str
    import_graph:   dict             # needed for import error context
    stage:          int | None       # stage 1/2/3 for coding errors


def save_repair_checkpoint(checkpoint: RepairCheckpoint, workspace_config: WorkspaceConfig):
    path = workspace_config.root_dir / "checkpoints" / "repair_state.json"
    data = {
        **asdict(checkpoint),
        "last_saved_at": time.time(),
        "paused_for_user": True,
    }
    # atomic write: write to temp file then rename
    tmp = path.with_suffix(".tmp")
    tmp.write_text(json.dumps(data, indent=2, default=str))
    tmp.replace(path)
```

### What Gets Saved at Each Escalation Point

| Escalation Point | Checkpoint Content |
|------------------|--------------------|
| Before blocking for user input | Full `RepairCheckpoint` + current file contents |
| After user responds "continue" | Updated `max_auto`, `attempt_count` |
| After user responds "skip" | `stubbed=True`, stub file written |
| After user provides hint | `user_hint` populated |
| After user signals "manual edit done" | File content at time of signal (re-checked after) |
| After each auto-repair attempt | `fix_history` appended, `attempt_count` updated |

### File Content Snapshot

Before any repair attempt (auto or user-guided), save a snapshot of the file:

```python
def save_file_snapshot(file_path: str, attempt_count: int, workspace_root: str):
    """
    Saves a numbered snapshot so we can show diff history to user.
    Saved to: {workspace_root}/checkpoints/snapshots/{rel_path}.attempt_{n}
    """
    rel = Path(file_path).relative_to(workspace_root)
    snap_dir = Path(workspace_root) / "checkpoints" / "snapshots" / rel.parent
    snap_dir.mkdir(parents=True, exist_ok=True)
    snap_path = snap_dir / f"{rel.name}.attempt_{attempt_count}"
    if Path(file_path).exists():
        snap_path.write_text(Path(file_path).read_text(encoding="utf-8"), encoding="utf-8")
```

### Resuming From Checkpoint After Restart

```python
def resume_repair_from_checkpoint(checkpoint_path: str) -> RepairCheckpoint:
    data = json.loads(Path(checkpoint_path).read_text())
    checkpoint = RepairCheckpoint(**data)

    print(f"Resuming repair for: {checkpoint.file_path}")
    print(f"  Attempts so far: {checkpoint.attempt_count}")
    print(f"  Last error: {checkpoint.last_error}")

    if data.get("paused_for_user"):
        # show the same escalation message again
        response = wait_for_escalation_response(
            EscalationContext.from_checkpoint(checkpoint)
        )
        return checkpoint, response

    return checkpoint, None
```

---

## 7. The "Fixable vs Human Judgment" Decision Tree

This decision tree determines whether to attempt auto-repair or escalate immediately.

```
INCOMING ERROR
      |
      +--[Level 5 LLM, transient]--------> exponential backoff retry (no escalation)
      |
      +--[Level 5 LLM, non-transient]----> IMMEDIATE ESCALATION
      |
      +--[Level 1/2/3/4 error]
                |
                v
         is_repair_stuck(fix_history)?
                |
                +--[YES]--------------> IMMEDIATE ESCALATION
                |                       (same error 3x in a row)
                |
                +--[NO]
                       |
                       v
                 error_class in IMMEDIATE_ESCALATE_CLASSES?
                 [FileNotFoundError for pipeline artifact,
                  RefusalError,
                  CircularImport after 2 attempts,
                  CUDA OOM with batch_size==1,
                  MemoryError with dataset at minimum]
                       |
                       +--[YES]-----> IMMEDIATE ESCALATION
                       |
                       +--[NO]
                              |
                              v
                       attempt_count <= max_auto?
                              |
                              +--[YES]-----> AUTO REPAIR ATTEMPT
                              |
                              +--[NO]------> SCHEDULED ESCALATION
                                            (budget exhausted)


AUTO REPAIR ATTEMPT:
  Is the error deterministically reproducible?
  (i.e., running checks again gives same error)
        |
        +--[NO: intermittent]----------> skip attempt, log warning, re-check
        |
        +--[YES]
                |
                v
          Is the error localized to this file?
          (no dependency on other files' content)
                |
                +--[YES]-----> Single-file patch repair
                |              (CoderAgent rewrites the failing function/class)
                |
                +--[NO]------> Multi-file repair
                               (CoderAgent considers import_graph context)
                               Higher token budget (REPAIR_CONTEXT_TOKENS)
```

---

## 8. Error Classification Implementation

```python
from enum import Enum
from dataclasses import dataclass

class ErrorType(Enum):
    SYNTAX          = "syntax"
    IMPORT          = "import"
    RUNTIME         = "runtime"
    RESULT          = "result"
    LLM             = "llm"
    WORKSPACE       = "workspace"   # e.g., file permission error
    UNKNOWN         = "unknown"

@dataclass
class CheckResult:
    ok:             bool
    error_type:     ErrorType | None = None
    error_class:    str = ""         # Python exception class name
    message:        str = ""
    line_number:    int | None = None
    col_offset:     int | None = None
    file_path:      str = ""
    extra:          dict = None
    warning:        str = ""         # non-fatal warning (e.g., IMPORT_SKIP)

    def __post_init__(self):
        if self.extra is None:
            self.extra = {}


# Master error classifier — entry point for all error routing
def classify_error(result: CheckResult) -> ErrorClassification:
    """
    Returns full classification with:
    - suggested_strategy: what repair to attempt
    - immediate_escalate: whether to skip auto-repair budget
    - context_needed: what information to gather before repair
    """
    strategies = {
        ErrorType.SYNTAX:   _classify_syntax,
        ErrorType.IMPORT:   _classify_import,
        ErrorType.RUNTIME:  _classify_runtime,
        ErrorType.RESULT:   _classify_result,
        ErrorType.LLM:      _classify_llm,
    }

    classifier = strategies.get(result.error_type, _classify_unknown)
    return classifier(result)


def _classify_import(result: CheckResult) -> ErrorClassification:
    ec = result.error_class
    msg = result.message

    if ec == "ModuleNotFoundError":
        missing = result.extra.get("missing_name", "")
        is_third_party = "." not in missing and missing not in sys.stdlib_module_names
        return ErrorClassification(
            strategy=RepairStrategy.PIP_THEN_REWRITE if is_third_party else RepairStrategy.FIX_SPELLING,
            immediate_escalate=False,
            context_needed=["import_graph", "requirements_txt"],
        )

    if ec == "CircularImportError":
        return ErrorClassification(
            strategy=RepairStrategy.LAZY_IMPORT,
            immediate_escalate=False,
            context_needed=["import_graph", "full_file_tree"],
            escalate_after_n=2,   # circular imports rarely fixed after 2 tries
        )

    return ErrorClassification(
        strategy=RepairStrategy.FIX_IMPORT_NAME,
        immediate_escalate=False,
        context_needed=["source_module_exports"],
    )
```

All error classifications flow through `classify_error()`. The repair loop calls this before each attempt to determine the correct repair strategy and gather necessary context. This ensures that repair prompts are always targeted, not generic.
