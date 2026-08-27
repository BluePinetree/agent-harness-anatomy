# AGENT_SPEC.md
# Agent Specifications — Role, Schema, Prompts, Tools, Failure Modes

> Version: 1.0  
> Last updated: 2026-05-19  
> Status: Authoritative specification for implementation

---

## Table of Contents

1. [WorkspaceSetupAgent](#1-workspacesetupagent)
2. [PlannerAgent](#2-planneragent)
3. [DesignerAgent](#3-designeragent)
4. [ApprovalGate](#4-approvalgate)
5. [CoderAgent](#5-coderagent)
6. [SyntaxChecker / ImportChecker](#6-syntaxchecker--importchecker)
7. [ExecutorAgent](#7-executoragent)
8. [AnalyzerAgent](#8-analyzeragent)
9. [WriterAgent](#9-writeragent)
10. [SelfVerifier](#10-selfverifier)
11. [IntegrationChecker](#11-integrationchecker)

---

## 1. WorkspaceSetupAgent

### Role
Not an LLM agent. Pure Python logic. Creates workspace directories, validates permissions, generates run_id, writes initial checkpoint.

### Input Schema

```python
from pydantic import BaseModel
from typing import Optional

class WorkspaceSetupInput(BaseModel):
    research_topic: str          # required
    workspace_path: Optional[str] = None  # optional override
```

### Output Schema

```python
from pathlib import Path

class WorkspaceConfig(BaseModel):
    root_dir:        Path
    run_id:          str       # 8-char hex UUID prefix
    session_id:      str       # full UUID4
    created_at:      float     # unix timestamp
    checkpoint_path: Path

    class Config:
        arbitrary_types_allowed = True
```

### Failure Modes

| Failure | Behavior |
|---------|----------|
| Path not writable | Raise `SetupError` — pipeline aborts (unrecoverable) |
| Disk full | Raise `SetupError` with disk usage info |
| Invalid topic string | Sanitize slug, never abort |

---

## 2. PlannerAgent

### Role
Designs the research plan given a topic. Produces a structured plan that drives all downstream agents. No tools needed — single LLM call with JSON output.

### Goal
Analyze the research topic and produce a precise, implementable research plan with clear success criteria and risk assessment.

### Backstory
You are a senior research scientist who specializes in designing reproducible ML experiments. You think in terms of falsifiable hypotheses, measurable metrics, and concrete baselines. You never propose experiments that cannot be implemented in a single Python script.

### Input Schema

```python
class PlannerInput(BaseModel):
    research_topic:       str
    workspace_config:     WorkspaceConfig
    user_feedback:        Optional[str] = None      # populated on REJECT/MODIFY cycles
    user_feedback_history: list[str] = []            # all prior feedback, oldest first
    attempt_number:       int = 1                    # which planning attempt this is
```

### Output Schema

```python
class PlannerResult(BaseModel):
    problem_statement:     str          # 1–2 sentences, precise
    research_questions:    list[str]    # 2–4 specific, answerable questions
    hypotheses:            list[str]    # one per research question
    success_criteria:      list[str]    # measurable, with numeric thresholds where possible
    baseline_comparisons:  list[str]    # what baselines to compare against
    recommended_profile:   str          # "classification" | "regression" | "generation" | etc.
    risk_items:            list[str]    # known implementation risks
    dataset_spec:          DatasetSpec
    algorithm_specs:       list[AlgorithmSpec]

class DatasetSpec(BaseModel):
    name:        str        # e.g., "CIFAR-10", "custom_synthetic"
    source:      str        # e.g., "torchvision.datasets", "sklearn.datasets"
    n_samples:   Optional[int]
    n_features:  Optional[int]
    n_classes:   Optional[int]
    train_test_split: float  # default 0.8

class AlgorithmSpec(BaseModel):
    name:         str
    role:         str     # "proposed" | "baseline"
    library:      str     # e.g., "torch", "sklearn", "xgboost"
    key_params:   dict    # e.g., {"lr": 0.001, "n_layers": 3}
```

### Task Prompt Template

```
You are a research planning specialist. Analyze the following research topic and produce a structured research plan.

RESEARCH TOPIC:
{research_topic}

{feedback_section}

Produce your response as a single JSON object with exactly these fields:
{{
  "problem_statement": "...",
  "research_questions": ["...", "..."],
  "hypotheses": ["...", "..."],
  "success_criteria": ["...", "..."],
  "baseline_comparisons": ["...", "..."],
  "recommended_profile": "...",
  "risk_items": ["...", "..."],
  "dataset_spec": {{
    "name": "...",
    "source": "...",
    "n_samples": ...,
    "n_features": ...,
    "n_classes": ...,
    "train_test_split": 0.8
  }},
  "algorithm_specs": [
    {{"name": "...", "role": "proposed", "library": "...", "key_params": {{...}}}},
    {{"name": "...", "role": "baseline", "library": "...", "key_params": {{...}}}}
  ]
}}

RULES:
1. Every algorithm must be implementable using only: torch, sklearn, xgboost, lightgbm, numpy, pandas, matplotlib.
2. Dataset must be available programmatically (no manual download required).
3. The experiment must complete within {MAX_EXPERIMENT_MINUTES} minutes on CPU.
4. Success criteria must include at least one numeric threshold (e.g., "accuracy >= 0.80").
5. Return ONLY the JSON object. No explanation text before or after.
```

### Feedback Section (populated when user_feedback is not None)

```
PREVIOUS PLAN WAS REJECTED/MODIFIED.
User feedback (most recent):
{user_feedback}

{prior_feedback_section}

Revise the plan to address this feedback. Keep what was not criticized.
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 2048 |
| temperature | 0.3 |
| max_retries | PLAN_MAX_RETRIES (3) on LLM error |

### Known Failure Modes

| Failure | Mitigation |
|---------|------------|
| LLM produces markdown-wrapped JSON | Strip ```json ... ``` wrappers before parsing |
| Missing required field in JSON | Pydantic validation error → re-prompt with field list |
| Dataset requires manual download | Risk item should note this; re-plan if user rejects |
| Algorithm uses unavailable library (e.g., TensorFlow) | Prompt explicitly lists allowed libraries |

---

## 3. DesignerAgent

### Role
Translates PlannerResult into a concrete file tree with dependency ordering, stage assignments, and import graph. No tools — single LLM call with JSON output.

### Goal
Design a minimal, complete file structure that can implement the research plan. Every file must have a clear purpose. Dependencies must be explicit. Files must be assignable to stages 1, 2, or 3 without circular dependencies between stages.

### Backstory
You are a software architect who designs Python research code. You believe in flat module hierarchies, explicit imports, and single-responsibility files. You never design a file without knowing exactly what it imports and what it exports.

### Input Schema

```python
class DesignerInput(BaseModel):
    planner_result:    PlannerResult
    workspace_config:  WorkspaceConfig
```

### Output Schema

```python
class FileSpec(BaseModel):
    path:           str             # relative path, e.g., "src/model.py"
    purpose:        str             # one sentence
    exports:        list[str]       # function/class names this file exports
    imports_from:   list[str]       # other workspace files this imports from (paths)
    stage:          int             # 1, 2, or 3
    is_mutable:     bool            # False for config/artifacts (read-only for coder)
    ast_outline:    list[str]       # e.g., ["class ResNet(nn.Module):", "def train():"]

class DesignerResult(BaseModel):
    file_tree:          list[FileSpec]
    generation_order:   list[str]       # dependency-sorted list of paths
    stage_assignments:  dict[str, int]  # path -> stage
    import_graph:       dict[str, list[str]]  # path -> list[paths it imports]
    entry_point:        str             # e.g., "src/main.py"
    success_criteria:   list[str]       # copied from PlannerResult, may be refined
    mutable_files:      list[str]       # paths CoderAgent must write
    readonly_files:     list[str]       # scaffold files written by system, not LLM
```

### Task Prompt Template

```
You are a Python software architect. Design a file structure that implements the following research plan.

RESEARCH PLAN:
{planner_result_json}

Design a file structure. Return a single JSON object with exactly these fields:
{{
  "file_tree": [
    {{
      "path": "src/config.py",
      "purpose": "Experiment configuration constants",
      "exports": ["CONFIG"],
      "imports_from": [],
      "stage": 1,
      "is_mutable": true,
      "ast_outline": ["CONFIG = {{'lr': 0.001, 'epochs': 10}}"]
    }},
    ...
  ],
  "generation_order": ["src/config.py", "src/data.py", ...],
  "stage_assignments": {{"src/config.py": 1, "src/model.py": 2, ...}},
  "import_graph": {{"src/model.py": ["src/config.py"], ...}},
  "entry_point": "src/main.py",
  "success_criteria": ["..."],
  "mutable_files": ["src/config.py", "src/model.py", ...],
  "readonly_files": ["src/artifacts.py", "src/config_schema.py"]
}}

RULES:
1. Stage 1: files with no imports from other workspace files (leaf nodes).
2. Stage 2: files that import from Stage 1 files only.
3. Stage 3: entry points and integration scripts (import from Stage 1 or 2).
4. Maximum 12 files total. Keep it minimal.
5. Use flat paths: "src/model.py" not "src/models/resnet/model.py".
6. Every file in mutable_files must appear in file_tree.
7. entry_point must be in Stage 3 and in mutable_files.
8. generation_order must be topologically sorted (no file before its dependencies).
9. Return ONLY the JSON object. No explanation text.
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 3000 |
| temperature | 0.2 |
| max_retries | DESIGN_MAX_RETRIES (3) on LLM error |

### Known Failure Modes

| Failure | Mitigation |
|---------|------------|
| Circular dependency in stage_assignments | Validate after parsing: topological sort; re-prompt if fails |
| Too many files (>12) | Prompt enforces limit; if exceeded, keep top-N by stage order |
| entry_point not in file_tree | Pydantic validator checks this |
| Missing imports in import_graph | Default to empty list; CoderAgent will discover actual imports |

---

## 4. ApprovalGate

### Role
Not an LLM agent. Synchronous I/O agent that blocks pipeline execution pending user input. Checkpoints state before blocking.

### Input Schema

```python
class ApprovalGateInput(BaseModel):
    planner_result:   PlannerResult
    designer_result:  DesignerResult
    workspace_config: WorkspaceConfig
    attempt_number:   int = 1
```

### Output Schema

```python
class ApprovalResponse(BaseModel):
    decision:     Literal["approve", "reject", "modify"]
    feedback:     Optional[str] = None    # text on reject or modify
    field_edits:  Optional[dict] = None   # on modify: {field_name: new_value}
    auto_approved: bool = False           # True if timeout triggered auto-approval
```

### Decision Routing

```python
def route_after_approval(response: ApprovalResponse) -> str:
    """Returns next pipeline node name."""
    if response.decision == "approve":
        return "coding"
    elif response.decision in ("reject", "modify"):
        return "planning"    # re-run PlannerAgent with feedback injected
    raise ValueError(f"Unknown decision: {response.decision}")
```

### Configuration

| Parameter | Value |
|-----------|-------|
| timeout_seconds | APPROVAL_TIMEOUT_SECONDS (3600) |
| max_approval_loops | APPROVAL_LOOP_MAX (10) |

---

## 5. CoderAgent

### Role
Writes Python source files to disk using tools. Operates in a `while(True) + tool_use` loop — never returns text output, only writes files. Has both a "write" mode (initial file creation) and a "repair" mode (targeted patch).

### Goal
Write exactly the files listed in your task. Each file must have correct Python syntax, be importable without errors, and implement the behavior described in the design spec. Use one tool call per turn. Never output code as text — always write to disk.

### Backstory
You are a Python engineer implementing a research experiment. You write clean, minimal, working code. When given a repair task, you make the smallest possible change to fix the error — you do not rewrite entire files unless necessary. You always verify your work by running syntax and import checks before reporting completion.

### Input Schema

```python
class CoderInput(BaseModel):
    designer_result:     DesignerResult
    workspace_config:    WorkspaceConfig
    stage:               int                 # 1, 2, or 3 (current stage)
    files_to_write:      list[FileSpec]      # files for this stage
    research_context:    str                 # PlannerResult summary for domain context
    repair_context:      Optional[RepairContext] = None   # populated in repair mode

class RepairContext(BaseModel):
    file_path:           str
    error_type:          str
    error_message:       str
    line_number:         Optional[int]
    fix_history:         list[dict]
    user_hint:           Optional[str]
    import_graph:        dict
    file_snapshot:       str             # current file content
```

### Output Schema

CoderAgent does not return an output schema — it writes files to disk. The caller verifies file existence after the tool loop completes.

```python
class CoderResult(BaseModel):
    files_written:    list[str]     # paths of files successfully written
    files_failed:     list[str]     # paths of files where checks failed
    tool_calls_made:  int
    turns_used:       int
```

### Tools

```python
CODER_TOOLS = [
    WorkspaceReadTool,       # read a file to understand existing code
    WorkspaceWriteTool,      # write (or append to) a file
    WorkspaceListTool,       # list directory contents
    SyntaxCheckTool,         # run ast.parse on a file
    ImportCheckTool,         # try to import a module
]
```

Tool schemas:

```python
# WorkspaceWriteTool
{
    "name": "workspace_write",
    "description": "Write or append content to a file. "
                   "mode='write' overwrites, mode='append' adds to end. "
                   "For files larger than ~300 lines, use multiple append calls.",
    "input_schema": {
        "type": "object",
        "properties": {
            "relative_path": {"type": "string", "description": "e.g., 'src/model.py'"},
            "content":       {"type": "string"},
            "mode":          {"type": "string", "enum": ["write", "append"], "default": "write"}
        },
        "required": ["relative_path", "content"]
    }
}

# SyntaxCheckTool
{
    "name": "syntax_check",
    "description": "Check Python syntax of a file using ast.parse. "
                   "Returns 'SYNTAX_OK' or error details with line number.",
    "input_schema": {
        "type": "object",
        "properties": {
            "relative_path": {"type": "string"}
        },
        "required": ["relative_path"]
    }
}

# ImportCheckTool
{
    "name": "import_check",
    "description": "Try to import a Python module. "
                   "Returns 'IMPORT_OK', 'IMPORT_SKIP' (DLL/OS error, not code issue), "
                   "or error details.",
    "input_schema": {
        "type": "object",
        "properties": {
            "relative_path": {"type": "string"}
        },
        "required": ["relative_path"]
    }
}
```

### Task Prompt Template — Write Mode

```
You are implementing a Python research experiment. Write the files listed below.

RESEARCH CONTEXT:
{research_context}

FILES TO WRITE (Stage {stage}):
{numbered_file_list}

For each file, the design specification is:
{file_specs_json}

IMPORT GRAPH (what each file imports):
{import_graph}

INSTRUCTIONS:
1. Write files ONE AT A TIME. Use workspace_write for each file.
2. After writing each file, call syntax_check on it.
3. After syntax_check passes, call import_check on it.
4. If either check fails, fix the file immediately before proceeding.
5. Do NOT write Final Answer until ALL files pass both checks.
6. Do NOT output code in text — only write to disk via workspace_write.
7. ONE tool call per turn. Never use arrays of tool calls.

CONSTRAINTS:
- Use only: {allowed_libraries}
- Do not use sklearn unless explicitly listed above.
- All file paths are relative to workspace root.
- Variable names match the ast_outline in the design spec.

Start with the first file. Write it now.
```

### Task Prompt Template — Repair Mode

```
You are repairing a Python file that failed checks.

FILE: {file_path}
ERROR TYPE: {error_type}
ERROR MESSAGE: {error_message}
{line_number_section}

CURRENT FILE CONTENT:
{file_snapshot}

REPAIR HISTORY (previous attempts that failed):
{fix_history_formatted}

{user_hint_section}

IMPORT GRAPH CONTEXT:
  {file_path} imports: {imports_from}
  {file_path} is imported by: {imported_by}

INSTRUCTIONS:
1. Read the current file with workspace_read if needed.
2. Make the MINIMUM change that fixes the error.
3. Write the fixed file with workspace_write.
4. Call syntax_check on the file.
5. Call import_check on the file.
6. If both pass: your work is done.
7. If checks still fail: make another targeted fix.
8. Do NOT output the file content as text. Only write to disk.
```

### User Hint Section (populated when user provided a hint)

```
USER FIX HINT:
"{user_hint}"

Incorporate this hint in your repair strategy.
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 8192 |
| temperature | 0.2 |
| max_turns_write | 40 (writing 12 files may need many turns) |
| max_turns_repair | 15 per file |

### Known Failure Modes

| Failure | Mitigation |
|---------|------------|
| LLM outputs code in text instead of writing | Prompt forbids this; if detected, extract and write programmatically |
| LLM writes multiple files in one tool call | Tool schema allows only one file; enforce server-side |
| LLM generates import for unlisted library | ImportCheckTool catches it; repair prompt re-states allowed libraries |
| File > 8192 tokens | Use append mode; split into sections |
| LLM stops before writing all files | Detect missing files after loop; re-enter with remaining file list |

---

## 6. SyntaxChecker / ImportChecker

### Role
Deterministic Python tools, not LLM agents. Used by CoderAgent as tools and also by the pipeline orchestrator after CoderAgent completes.

### SyntaxCheckTool Implementation

```python
import ast
from pathlib import Path

class SyntaxCheckTool:
    name = "syntax_check"

    def run(self, workspace_root: str, relative_path: str) -> str:
        path = Path(workspace_root) / relative_path
        if not path.exists():
            return f"SYNTAX_ERROR: File not found: {relative_path}"
        try:
            source = path.read_text(encoding="utf-8")
            ast.parse(source)
            return f"SYNTAX_OK: {relative_path}"
        except SyntaxError as e:
            return (
                f"SYNTAX_ERROR: {e.__class__.__name__} at line {e.lineno}: {e.msg}\n"
                f"  File: {relative_path}\n"
                f"  Line content: {e.text or '(unavailable)'}"
            )
        except Exception as e:
            return f"SYNTAX_ERROR: Unexpected: {e}"
```

### ImportCheckTool Implementation

```python
import subprocess, sys

IMPORT_SKIP_PATTERNS = [
    r"DLL load failed",
    r"cannot open shared object",
    r"The specified module could not be found",
]

class ImportCheckTool:
    name = "import_check"

    def run(self, workspace_root: str, relative_path: str) -> str:
        module = (
            Path(relative_path)
            .with_suffix("")
            .as_posix()
            .replace("/", ".")
        )
        cmd = [
            sys.executable, "-c",
            f"import sys; sys.path.insert(0, '{workspace_root}'); import {module}"
        ]
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)

        if result.returncode == 0:
            return f"IMPORT_OK: {module}"

        stderr = result.stderr
        for pattern in IMPORT_SKIP_PATTERNS:
            if re.search(pattern, stderr, re.IGNORECASE):
                return f"IMPORT_SKIP: DLL/OS error (not a code issue): {stderr[:200]}"

        return f"IMPORT_ERROR: {stderr[-1000:]}"
```

---

## 7. ExecutorAgent

### Role
Runs the experiment entry point, captures all output, and collects artifacts. Operates via tool loop — does not generate code, only runs it.

### Goal
Execute the experiment exactly as designed. Report actual results only — never fabricate metrics. If execution fails, report the exact error so the repair loop can proceed.

### Backstory
You are a lab technician running an experiment. Your job is to press the start button, watch what happens, and write down exactly what you saw — including failures. You do not interpret results, invent numbers, or modify the experiment. You report return codes, stdout, stderr, and whether result.json was written.

### Input Schema

```python
class ExecutorInput(BaseModel):
    designer_result:   DesignerResult
    workspace_config:  WorkspaceConfig
    exec_attempt:      int = 1
    prior_stderr:      Optional[str] = None   # from prior failed attempt
```

### Output Schema

```python
class ExecutorResult(BaseModel):
    success:         bool
    return_code:     int        # -2 if timeout/crash before exit
    duration_sec:    float
    stdout_tail:     str        # last 2000 chars
    stderr_tail:     str        # last 2000 chars
    result_json:     Optional[dict]          # parsed result.json, or None
    artifact_paths:  list[str]               # all files written in results/, plots/
    metrics:         dict                    # extracted from result_json.metrics
    repair_count:    int = 0
    stubbed_files:   list[str] = []
```

### Tools

```python
EXECUTOR_TOOLS = [
    RunCommandTool,      # execute shell command, capture output
    ReadResultTool,      # read and parse results/result.json
    WorkspaceListTool,   # list artifact files
]
```

```python
# RunCommandTool
{
    "name": "run_command",
    "description": "Run a shell command in the workspace directory. "
                   "Returns: return_code, stdout (truncated), stderr (truncated), duration_sec. "
                   "NEVER fabricate output. Report exactly what the command produced.",
    "input_schema": {
        "type": "object",
        "properties": {
            "command":     {"type": "string"},
            "working_dir": {"type": "string"},
            "timeout_sec": {"type": "number", "default": 300}
        },
        "required": ["command", "working_dir"]
    }
}
```

### Task Prompt Template

```
You are running a research experiment. Execute the entry point and report results.

WORKSPACE: {workspace_root}
ENTRY POINT: {entry_point}
EXPERIMENT ID: {run_id}

INSTRUCTIONS:
1. Call run_command with:
   command="{python_executable} {entry_point}"
   working_dir="{workspace_root}"
   timeout_sec={EXEC_TIMEOUT_SECONDS}
   
2. Record the EXACT return_code, stdout_tail, and stderr_tail.
   DO NOT modify, interpret, or fabricate these values.
   If return_code == -2, that means the process was killed (timeout or crash).

3. If return_code == 0:
   Call read_result to check if results/result.json was written.
   If result.json exists, your task is complete.
   
4. If return_code != 0 OR result.json is missing:
   Report: return_code={actual_code}, the stderr content, and
   "EXECUTION_FAILED: result.json not found" if applicable.
   DO NOT attempt to fix the code yourself.
   DO NOT run the command again without being instructed to.

5. Call workspace_list to collect artifact paths from results/ and plots/.

Your FINAL ANSWER must be a JSON object:
{{
  "success": true/false,
  "return_code": {actual_return_code},
  "duration_sec": {actual_duration},
  "stdout_tail": "...",
  "stderr_tail": "...",
  "result_json": {{...}} or null,
  "artifact_paths": ["..."],
  "metrics": {{...}}
}}
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 2048 |
| temperature | 0.0 (deterministic) |
| max_turns | 5 |

### Known Failure Modes

| Failure | Mitigation |
|---------|------------|
| LLM fabricates return_code | Temperature=0 + explicit no-fabricate instruction + tool result is ground truth |
| LLM re-runs experiment without being asked | Prompt forbids this explicitly |
| Timeout not respected | RunCommandTool uses `subprocess.run(timeout=N)` Python-side |
| Large stdout overflows context | tool_result_budget truncates to 2000 chars |

---

## 8. AnalyzerAgent

### Role
Analyzes experiment results, computes derived metrics, and decides whether the experiment meets the success criteria. Single LLM call with structured output.

### Goal
Interpret the experimental results objectively. Check each success criterion against actual metrics. Do not judge the quality of the algorithm — only judge whether the criteria were met. Produce a structured analysis that the WriterAgent can use.

### Input Schema

```python
class AnalyzerInput(BaseModel):
    executor_result:   ExecutorResult
    planner_result:    PlannerResult
    workspace_config:  WorkspaceConfig
```

### Output Schema

```python
class AnalyzerResult(BaseModel):
    meets_criteria:      bool
    criteria_details:    list[CriterionResult]
    key_findings:        list[str]       # 3–5 bullet points for WriterAgent
    comparison_table:    list[dict]      # rows: {method, metric, value}
    statistical_notes:   list[str]       # any significance/variance observations
    suggested_discussion: list[str]      # insights for Discussion/Conclusion

class CriterionResult(BaseModel):
    criterion:    str
    met:          bool
    actual_value: Optional[float]
    threshold:    Optional[float]
    note:         str
```

### Task Prompt Template

```
Analyze the following experiment results against the success criteria.

SUCCESS CRITERIA:
{success_criteria_numbered}

EXPERIMENT RESULTS:
{executor_result_json}

For each criterion, determine if it was met based on the actual metrics.
Return a JSON object:
{{
  "meets_criteria": true/false,
  "criteria_details": [
    {{"criterion": "...", "met": true/false, "actual_value": ..., "threshold": ..., "note": "..."}}
  ],
  "key_findings": ["...", "..."],
  "comparison_table": [
    {{"method": "...", "metric": "...", "value": ...}}
  ],
  "statistical_notes": ["..."],
  "suggested_discussion": ["..."]
}}

RULES:
1. Only use numbers that appear in the EXPERIMENT RESULTS above.
2. If a metric is missing from results, set actual_value to null and met to false.
3. Do not speculate about why results are good or bad.
4. Return ONLY the JSON object.
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 2048 |
| temperature | 0.1 |

---

## 9. WriterAgent

### Role
Writes one paper section at a time. Uses context_bank (prior written sections) and ExecutorResult for factual grounding. Operates section-by-section in a tool loop.

### Goal
Write each paper section based on actual experimental results. Every factual claim must be traceable to the ExecutorResult. Maintain coherence with previously written sections. Write in formal academic English.

### Backstory
You are an academic writer with expertise in machine learning research papers. You write clearly and precisely. You cite only results that actually occurred. You maintain consistent terminology across sections. You know that the Experiments section is the anchor — every other section frames or discusses what actually happened.

### Input Schema

```python
class WriterInput(BaseModel):
    section_name:       str             # e.g., "Experiments"
    section_index:      int             # position in write order
    executor_result:    ExecutorResult
    analyzer_result:    AnalyzerResult
    planner_result:     PlannerResult
    context_bank:       dict[str, str]  # prior written sections {name: text}
    workspace_config:   WorkspaceConfig
    output_format:      Literal["markdown", "latex"] = "markdown"
    revision_feedback:  Optional[str] = None  # from SelfVerifier on revision
```

### Output Schema

```python
class SectionResult(BaseModel):
    section_name:    str
    text:            str
    word_count:      int
    citation_count:  int
    needs_review:    bool = False
    quality_score:   float          # 0.0–1.0, computed by SelfVerifier
    revision_count:  int
```

### Task Prompt Template — Write Mode

```
Write the {section_name} section of a research paper.

PAPER FORMAT: {output_format}

RESEARCH CONTEXT:
  Topic: {research_topic}
  Problem: {problem_statement}

EXPERIMENTAL RESULTS (use only these numbers in your writing):
{executor_result_json}

ANALYSIS FINDINGS:
{analyzer_result_json}

PRIOR SECTIONS WRITTEN (for coherence):
{context_bank_summary}

SECTION-SPECIFIC INSTRUCTIONS:
{section_instructions}

REQUIREMENTS:
1. Minimum {min_words} words.
2. Reference at least {min_citations} citations in the form [N] or \cite{{key}}.
3. Use only result numbers that appear in EXPERIMENTAL RESULTS above.
4. Maintain consistent terminology with prior sections.
5. Do not begin with "In this section" or "This section discusses".
6. Write in formal academic English.

Write the section now. Output only the section text, no meta-commentary.
```

### Section-Specific Instructions

```python
SECTION_INSTRUCTIONS = {
    "Experiments": """
        Structure:
        1. Experimental Setup (dataset, hardware, hyperparameters)
        2. Results Table (comparison of all methods on all metrics)
        3. Analysis of Results (what the numbers mean)
        
        REQUIRED: Include a Markdown/LaTeX table with all comparison_table rows.
        REQUIRED: State whether each success criterion was met.
        Reference exact metric values from the ExecutorResult.
    """,

    "Proposed_Method": """
        Structure:
        1. Motivation (why this approach)
        2. Architecture/Algorithm Description
        3. Key Design Decisions (reference the algorithm_specs)
        4. Training Procedure
        
        Use pseudocode or equations if the format supports it.
        Do not reference results — those go in Experiments.
    """,

    "Introduction": """
        Structure:
        1. Problem motivation (2–3 sentences)
        2. Limitations of existing approaches (reference Related Works section)
        3. Our contribution (what we propose)
        4. Paper organization (one sentence per section)
        
        Reference the research questions from the plan.
        Do not reveal specific result numbers (save for Experiments).
    """,

    "Related_Works": """
        Discuss existing approaches relevant to:
        {baseline_comparisons}
        
        Group into 2–3 thematic clusters.
        Each cluster: 2–4 citations.
        End with: how our work differs from each cluster.
        
        Use placeholder citations if exact papers are unknown: [AuthorYear].
    """,

    "Conclusion": """
        Structure:
        1. Summary of what was done (1 paragraph)
        2. Key findings (reference specific metric values)
        3. Limitations
        4. Future work
        
        Do not introduce new information.
        Reference the success criteria and whether they were met.
    """,

    "References": """
        List all citations used in the paper.
        Format: [N] Author(s). Title. Venue, Year.
        
        Include at least {min_references} entries.
        Use consistent formatting.
        Order by citation number.
    """,

    "Abstract": """
        Structure (one paragraph, {min_words}–300 words):
        1. Context (1 sentence)
        2. Problem (1 sentence)
        3. Proposed approach (1–2 sentences)
        4. Key results (2–3 sentences with actual numbers)
        5. Conclusion (1 sentence)
        
        This is the last section written. All results are known.
        Include the most important metric values.
    """,
}
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 4096 |
| temperature | 0.4 |
| max_turns | 3 (writing is single-shot with optional revision) |

### Tools

WriterAgent uses `WriteSectionTool` to write its section to disk:

```python
{
    "name": "write_section",
    "description": "Write a paper section to the paper directory.",
    "input_schema": {
        "type": "object",
        "properties": {
            "section_name": {"type": "string"},
            "content":      {"type": "string"},
            "mode":         {"type": "string", "enum": ["write", "append"]}
        },
        "required": ["section_name", "content"]
    }
}
```

### Known Failure Modes

| Failure | Mitigation |
|---------|------------|
| LLM invents metric values | SelfVerifier cross-checks against ExecutorResult |
| Section too short | SelfVerifier enforces MIN_WORDS per section |
| Inconsistent terminology | context_bank provides prior sections for reference |
| Unclosed LaTeX environments | SelfVerifier checks LaTeX syntax |

---

## 10. SelfVerifier

### Role
Deterministic checker that validates each written section. Not an LLM — uses regex patterns, word counting, and metric cross-referencing.

### Input Schema

```python
class VerifierInput(BaseModel):
    section_name:    str
    section_text:    str
    executor_result: ExecutorResult
    output_format:   Literal["markdown", "latex"]
```

### Output Schema

```python
class VerifyResult(BaseModel):
    ok:              bool
    score:           float          # 0.0–1.0 weighted average of sub-checks
    failures:        list[str]      # human-readable failure descriptions
    suggestions:     list[str]      # for WriterAgent revision prompt

class VerifyCheckResult(BaseModel):
    check_name:    str
    passed:        bool
    weight:        float            # contribution to overall score
    message:       str
```

### Check Weights

```python
CHECK_WEIGHTS = {
    "min_words":          0.30,
    "result_references":  0.35,   # highest weight — factual grounding is critical
    "citations":          0.20,
    "syntax":             0.15,
}
```

### Score Computation

```python
def compute_score(check_results: list[VerifyCheckResult]) -> float:
    total_weight = sum(c.weight for c in check_results)
    weighted_pass = sum(c.weight for c in check_results if c.passed)
    return weighted_pass / total_weight if total_weight > 0 else 0.0
```

### Revision Prompt Construction

When `SelfVerifier` fails, it constructs a targeted revision prompt:

```python
def build_revision_prompt(failures: list[str], suggestions: list[str]) -> str:
    return (
        "The previous draft of this section failed quality checks:\n\n"
        + "\n".join(f"- {f}" for f in failures)
        + "\n\nSuggestions for revision:\n"
        + "\n".join(f"- {s}" for s in suggestions)
        + "\n\nRevise the section to address all failures above. "
          "Keep the parts that passed. Return only the revised section text."
    )
```

---

## 11. IntegrationChecker

### Role
After all sections are written, checks cross-section coherence. Single LLM call with structured output.

### Input Schema

```python
class IntegrationInput(BaseModel):
    sections:          dict[str, str]     # section_name -> text
    executor_result:   ExecutorResult
    analyzer_result:   AnalyzerResult
```

### Output Schema

```python
class IntegrationResult(BaseModel):
    overall_score:       float              # 0.0–1.0
    coherence_issues:    list[CoherenceIssue]
    sections_to_rewrite: list[str]          # sections with score < SECTION_REWRITE_THRESHOLD

class CoherenceIssue(BaseModel):
    issue_type:    str    # "terminology_mismatch", "contradictory_claim", "missing_reference"
    section_a:     str
    section_b:     str
    description:   str
    severity:      Literal["low", "medium", "high"]
```

### Task Prompt Template

```
Review these paper sections for cross-section coherence. Identify contradictions,
terminology mismatches, and missing references between sections.

{sections_json}

FACTUAL GROUND TRUTH:
{executor_result_json}

Identify ALL of the following issues:
1. Does Proposed_Method describe the same algorithm that Experiments ran?
2. Does Conclusion cite results that appear in Experiments?
3. Does Introduction's claimed contribution match what Proposed_Method describes?
4. Are there any numbers in any section that contradict the experimental results?
5. Is technical terminology consistent across all sections?

Return a JSON object:
{{
  "overall_score": 0.0–1.0,
  "coherence_issues": [
    {{
      "issue_type": "...",
      "section_a": "...",
      "section_b": "...",
      "description": "...",
      "severity": "low/medium/high"
    }}
  ],
  "sections_to_rewrite": ["..."]
}}

Be specific. Reference actual text passages when describing issues.
```

### Configuration

| Parameter | Value |
|-----------|-------|
| model | claude-sonnet-4-6 |
| max_tokens | 3000 |
| temperature | 0.1 |
