# CONSTANTS.md
# All Configurable Constants — Defaults, Ranges, Behavior, and Tuning Guide

> Version: 1.0  
> Last updated: 2026-05-19  
> Status: Authoritative specification for implementation

All constants are centralized in `platform/constants.py`. They can be overridden via environment variables or a `config.yaml` file. Environment variable names follow the pattern `RESEARCH_PIPELINE_{CONSTANT_NAME}`.

---

## Table of Contents

1. [constants.py — Master File](#1-constantspy--master-file)
2. [Phase 0: Workspace](#2-phase-0-workspace)
3. [Phase 1: Planning and Design](#3-phase-1-planning-and-design)
4. [Phase 2: Staged Coding](#4-phase-2-staged-coding)
5. [Phase 3: Execution](#5-phase-3-execution)
6. [Phase 4: Paper Writing](#6-phase-4-paper-writing)
7. [LLM and API](#7-llm-and-api)
8. [Tool Result Budgets](#8-tool-result-budgets)
9. [Token Budgets](#9-token-budgets)
10. [User Interaction](#10-user-interaction)
11. [Tuning Guide by Scenario](#11-tuning-guide-by-scenario)

---

## 1. constants.py — Master File

```python
"""
platform/constants.py
All configurable constants for the research pipeline.
Override via environment variable: RESEARCH_PIPELINE_{NAME}
or via config.yaml key matching the snake_case constant name.
"""

import os
from typing import Any

def _env(name: str, default: Any, cast=str) -> Any:
    """Read constant from environment, with type casting."""
    raw = os.environ.get(f"RESEARCH_PIPELINE_{name}", None)
    if raw is None:
        return default
    try:
        return cast(raw)
    except (ValueError, TypeError):
        return default

# ============================================================
# PHASE 0: WORKSPACE
# ============================================================

# Default output directory (relative to cwd)
WORKSPACE_DEFAULT_ROOT = _env("WORKSPACE_DEFAULT_ROOT", "outputs", str)

# Subdirectories to create under workspace root
WORKSPACE_SUBDIRS = ["src", "results", "logs", "plots", "checkpoints", "paper"]

# ============================================================
# PHASE 1: PLANNING AND DESIGN
# ============================================================

# Max LLM-error retries for PlannerAgent
PLAN_MAX_RETRIES = _env("PLAN_MAX_RETRIES", 3, int)
# range: [1, 10]; too high wastes budget on broken prompts

# Max LLM-error retries for DesignerAgent
DESIGN_MAX_RETRIES = _env("DESIGN_MAX_RETRIES", 3, int)
# range: [1, 10]

# Max total approval-loop cycles (reject/modify before force-approve)
APPROVAL_LOOP_MAX = _env("APPROVAL_LOOP_MAX", 10, int)
# range: [1, 100]; set to 1 for non-interactive mode

# Timeout for approval gate (seconds); 0 = never timeout (block forever)
APPROVAL_TIMEOUT_SECONDS = _env("APPROVAL_TIMEOUT_SECONDS", 3600, float)
# range: [0, 86400]; 3600 = 1 hour; 0 = infinite wait

# Max files in DesignerResult.file_tree
DESIGNER_MAX_FILES = _env("DESIGNER_MAX_FILES", 12, int)
# range: [3, 30]; more files = longer coding phase

# ============================================================
# PHASE 2: STAGED CODING
# ============================================================

# Max automatic repair attempts per file before escalating to user
MAX_AUTO_ATTEMPTS = _env("MAX_AUTO_ATTEMPTS", 5, int)
# range: [1, 20]
# effect: higher = more LLM API calls before human is involved
# tune: increase if LLMs are capable but need more tries;
#       decrease for faster human involvement

# Extra attempts granted when user chooses "continue"
DEFAULT_EXTRA_ATTEMPTS = _env("DEFAULT_EXTRA_ATTEMPTS", 3, int)
# range: [1, 10]

# Hint-guided repair: extra attempts given when user provides a hint
HINT_EXTRA_ATTEMPTS = _env("HINT_EXTRA_ATTEMPTS", 3, int)
# range: [1, 10]

# Number of identical consecutive errors that trigger "stuck" detection
STUCK_LOOP_THRESHOLD = _env("STUCK_LOOP_THRESHOLD", 3, int)
# range: [2, 10]; lower = faster escalation when LLM is looping
# effect: if the last N repairs produced the same error, escalate immediately

# Max tool_loop turns for CoderAgent in write mode
CODER_MAX_TURNS_WRITE = _env("CODER_MAX_TURNS_WRITE", 40, int)
# range: [10, 100]; writing 12 files may require 40+ turns

# Max tool_loop turns for CoderAgent in repair mode (per file)
CODER_MAX_TURNS_REPAIR = _env("CODER_MAX_TURNS_REPAIR", 15, int)
# range: [5, 30]

# After how many auto-repair attempts does CircularImportError escalate
CIRCULAR_IMPORT_ESCALATE_AFTER = _env("CIRCULAR_IMPORT_ESCALATE_AFTER", 2, int)
# range: [1, 5]; circular imports are structural and rarely fixed by LLM alone

# ============================================================
# PHASE 3: EXECUTION
# ============================================================

# Max time in seconds for experiment execution
EXEC_TIMEOUT_SECONDS = _env("EXEC_TIMEOUT_SECONDS", 300, float)
# range: [30, 7200]
# effect: too low = timeout on legitimate long experiments;
#         too high = runaway experiments block pipeline
# tune: set based on expected experiment size (small: 300, medium: 900, large: 3600)

# Max automatic execution repair attempts before escalating
MAX_EXEC_ATTEMPTS = _env("MAX_EXEC_ATTEMPTS", 3, int)
# range: [1, 10]

# Max automatic result-repair attempts before escalating
MAX_RESULT_REPAIR_ATTEMPTS = _env("MAX_RESULT_REPAIR_ATTEMPTS", 2, int)
# range: [1, 5]

# Reduced dataset size for MemoryError recovery
REDUCED_DATASET_SIZE = _env("REDUCED_DATASET_SIZE", 1000, int)
# range: [100, 100000]

# Maximum experiment execution time used in planner prompt
MAX_EXPERIMENT_MINUTES = _env("MAX_EXPERIMENT_MINUTES", 5, int)
# range: [1, 120]; tells PlannerAgent to design accordingly

# Minimum dataset size below which we escalate instead of reducing further
MIN_DATASET_SIZE = _env("MIN_DATASET_SIZE", 100, int)
# range: [10, 10000]

# Minimum batch size below which we escalate instead of reducing further (GPU)
MIN_BATCH_SIZE = _env("MIN_BATCH_SIZE", 1, int)
# range: [1, 16]; 1 = cannot reduce further → escalate

# ============================================================
# PHASE 4: PAPER WRITING
# ============================================================

# Max revision cycles per section before marking NEEDS_REVIEW
MAX_SECTION_REVISIONS = _env("MAX_SECTION_REVISIONS", 3, int)
# range: [1, 10]

# Overall paper quality score threshold (below = targeted re-write)
QUALITY_THRESHOLD = _env("QUALITY_THRESHOLD", 0.75, float)
# range: [0.0, 1.0]; 0.75 = 75% of checks must pass

# Per-section score threshold (below = added to sections_to_rewrite list)
SECTION_REWRITE_THRESHOLD = _env("SECTION_REWRITE_THRESHOLD", 0.60, float)
# range: [0.0, 1.0]

# Minimum word counts per section
MIN_WORDS_ABSTRACT         = _env("MIN_WORDS_ABSTRACT",         150, int)
MIN_WORDS_INTRODUCTION     = _env("MIN_WORDS_INTRODUCTION",     400, int)
MIN_WORDS_RELATED_WORKS    = _env("MIN_WORDS_RELATED_WORKS",    400, int)
MIN_WORDS_PROPOSED_METHOD  = _env("MIN_WORDS_PROPOSED_METHOD",  500, int)
MIN_WORDS_EXPERIMENTS      = _env("MIN_WORDS_EXPERIMENTS",      600, int)
MIN_WORDS_CONCLUSION       = _env("MIN_WORDS_CONCLUSION",       200, int)

MIN_WORDS_BY_SECTION = {
    "Abstract":        MIN_WORDS_ABSTRACT,
    "Introduction":    MIN_WORDS_INTRODUCTION,
    "Related_Works":   MIN_WORDS_RELATED_WORKS,
    "Proposed_Method": MIN_WORDS_PROPOSED_METHOD,
    "Experiments":     MIN_WORDS_EXPERIMENTS,
    "Conclusion":      MIN_WORDS_CONCLUSION,
    "References":      0,
}

# Minimum reference entries in References section
MIN_REFERENCE_ENTRIES = _env("MIN_REFERENCE_ENTRIES", 5, int)
# range: [3, 50]

# Max chars from context_bank passed to each WriterAgent call
CONTEXT_BANK_MAX_CHARS = _env("CONTEXT_BANK_MAX_CHARS", 3000, int)
# range: [500, 10000]; larger = better coherence but more tokens per call

# Write order for paper sections (topological: facts before framing)
PAPER_WRITE_ORDER = [
    "Experiments",       # written first: anchors factual content
    "Proposed_Method",   # describes what was implemented
    "Introduction",      # motivates the work
    "Related_Works",     # positions the work
    "Conclusion",        # summarizes findings
    "References",        # bibliography
    "Abstract",          # written last: summarizes the whole paper
]

# Final paper read order (standard academic order)
PAPER_READ_ORDER = [
    "Abstract",
    "Introduction",
    "Related_Works",
    "Proposed_Method",
    "Experiments",
    "Conclusion",
    "References",
]

# ============================================================
# LLM AND API
# ============================================================

# Default model
LLM_MODEL = _env("LLM_MODEL", "claude-sonnet-4-6", str)

# Max LLM retries for transient errors (rate limit, timeout, overload)
MAX_LLM_RETRIES = _env("MAX_LLM_RETRIES", 10, int)
# range: [3, 20]

# Exponential backoff base delay (seconds)
BACKOFF_BASE_SEC = _env("BACKOFF_BASE_SEC", 0.5, float)
# range: [0.1, 5.0]

# Exponential backoff max delay (seconds)
BACKOFF_MAX_SEC = _env("BACKOFF_MAX_SEC", 32.0, float)
# range: [5.0, 120.0]

# Jitter fraction (0.25 = up to 25% random jitter added to delay)
BACKOFF_JITTER_FRACTION = _env("BACKOFF_JITTER_FRACTION", 0.25, float)
# range: [0.0, 0.5]

# Per-agent max_tokens settings
MAX_TOKENS_PLANNER    = _env("MAX_TOKENS_PLANNER",    2048, int)
MAX_TOKENS_DESIGNER   = _env("MAX_TOKENS_DESIGNER",   3000, int)
MAX_TOKENS_CODER      = _env("MAX_TOKENS_CODER",      8192, int)
MAX_TOKENS_EXECUTOR   = _env("MAX_TOKENS_EXECUTOR",   2048, int)
MAX_TOKENS_ANALYZER   = _env("MAX_TOKENS_ANALYZER",   2048, int)
MAX_TOKENS_WRITER     = _env("MAX_TOKENS_WRITER",     4096, int)
MAX_TOKENS_INTEGRATION = _env("MAX_TOKENS_INTEGRATION", 3000, int)

# Per-agent temperature settings
TEMPERATURE_PLANNER   = _env("TEMPERATURE_PLANNER",   0.3,  float)
TEMPERATURE_DESIGNER  = _env("TEMPERATURE_DESIGNER",  0.2,  float)
TEMPERATURE_CODER     = _env("TEMPERATURE_CODER",     0.2,  float)
TEMPERATURE_EXECUTOR  = _env("TEMPERATURE_EXECUTOR",  0.0,  float)  # fully deterministic
TEMPERATURE_ANALYZER  = _env("TEMPERATURE_ANALYZER",  0.1,  float)
TEMPERATURE_WRITER    = _env("TEMPERATURE_WRITER",    0.4,  float)

# ============================================================
# TOOL RESULT BUDGETS
# ============================================================

# Max chars per tool result before truncation
MAX_TOOL_RESULT_CHARS = _env("MAX_TOOL_RESULT_CHARS", 4000, int)
# range: [500, 20000]
# effect: larger = more context for LLM, but burns more tokens
# tune: increase if agents miss important context from tool results

# Max chars of stdout/stderr kept in ExecutorResult
MAX_OUTPUT_TAIL_CHARS = _env("MAX_OUTPUT_TAIL_CHARS", 2000, int)
# range: [200, 10000]

# Max chars of stderr shown in escalation message to user
MAX_STDERR_DISPLAY_LINES = _env("MAX_STDERR_DISPLAY_LINES", 50, int)
# range: [10, 200]

# ============================================================
# TOKEN BUDGETS (Diminishing Returns Detection)
# ============================================================

# Context window fraction that triggers near-limit warning
CONTEXT_NEAR_LIMIT_FRACTION = _env("CONTEXT_NEAR_LIMIT_FRACTION", 0.90, float)
# range: [0.70, 0.95]

# Consecutive turns with output tokens below this = stuck/looping
DIMINISHING_RETURNS_THRESHOLD = _env("DIMINISHING_RETURNS_THRESHOLD", 500, int)
# range: [100, 2000]

# Number of consecutive low-delta turns before declaring diminishing returns
DIMINISHING_RETURNS_WINDOW = _env("DIMINISHING_RETURNS_WINDOW", 3, int)
# range: [2, 6]

# ============================================================
# USER INTERACTION
# ============================================================

# Minimum seconds between escalation notifications (prevents spam)
ESCALATION_RATE_LIMIT_SEC = _env("ESCALATION_RATE_LIMIT_SEC", 5.0, float)
# range: [0, 60]

# Number of repair attempts summarized in escalation message
ESCALATION_HISTORY_DISPLAY = _env("ESCALATION_HISTORY_DISPLAY", 5, int)
# range: [1, 20]; how many prior attempts to show the user

# Timeout for user to respond to escalation (seconds); 0 = block forever
ESCALATION_TIMEOUT_SECONDS = _env("ESCALATION_TIMEOUT_SECONDS", 0, float)
# range: [0, 86400]; 0 = block forever (recommended for interactive use)

# ============================================================
# ALLOWED LIBRARIES (for CoderAgent prompt)
# ============================================================

ALLOWED_LIBRARIES = [
    "torch", "torchvision",
    "numpy", "pandas",
    "matplotlib", "seaborn",
    "xgboost", "lightgbm",
    "scikit-learn",
    "scipy",
    "json", "pathlib", "os", "sys", "time", "random",
    "dataclasses", "typing", "abc", "collections",
    "argparse", "logging",
]
# Note: CoderAgent prompt lists these explicitly.
# "sklearn" is an alias — use "scikit-learn" for pip, "sklearn" for import.
```

---

## 2. Phase 0: Workspace

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `WORKSPACE_DEFAULT_ROOT` | `"outputs"` | any valid path | Base directory for all runs | Change if running in a container or on a cluster with a specific output mount |
| `WORKSPACE_SUBDIRS` | `[src, results, logs, plots, checkpoints, paper]` | — | Directories created on startup | Add `"models"` if checkpointing model weights |

---

## 3. Phase 1: Planning and Design

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `PLAN_MAX_RETRIES` | `3` | [1, 10] | How many times PlannerAgent retries on LLM error before aborting | Increase if hitting intermittent API errors; decrease in batch mode |
| `DESIGN_MAX_RETRIES` | `3` | [1, 10] | Same for DesignerAgent | Same as above |
| `APPROVAL_LOOP_MAX` | `10` | [1, 100] | Max plan re-generations before forcing override | Set to 1 for fully automated pipelines; set to 100 for research use |
| `APPROVAL_TIMEOUT_SECONDS` | `3600` | [0, 86400] | How long to wait for user approval | Set to 0 for interactive terminals; set to 7200 for overnight batch |
| `DESIGNER_MAX_FILES` | `12` | [3, 30] | Maximum files DesignerAgent can propose | Increase for complex experiments; decrease for quick prototypes |

---

## 4. Phase 2: Staged Coding

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `MAX_AUTO_ATTEMPTS` | `5` | [1, 20] | LLM auto-repair budget per file | Increase for weak models that need more iterations; decrease for expensive API usage |
| `DEFAULT_EXTRA_ATTEMPTS` | `3` | [1, 10] | Extra attempts granted on user "continue" | Increase if users frequently grant more budget |
| `HINT_EXTRA_ATTEMPTS` | `3` | [1, 10] | Extra attempts after user provides a hint | Usually matches DEFAULT_EXTRA_ATTEMPTS |
| `STUCK_LOOP_THRESHOLD` | `3` | [2, 10] | Identical consecutive errors triggering immediate escalation | Lower = faster human escalation; higher = more trust in LLM variation |
| `CODER_MAX_TURNS_WRITE` | `40` | [10, 100] | Max tool_loop turns for CoderAgent writing all stage files | Increase if writing many files (>10) or large files |
| `CODER_MAX_TURNS_REPAIR` | `15` | [5, 30] | Max turns for one file repair | Increase for complex structural repairs |
| `CIRCULAR_IMPORT_ESCALATE_AFTER` | `2` | [1, 5] | Attempts before circular import escalates | 2 is almost always correct — circular imports need structural changes |

---

## 5. Phase 3: Execution

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `EXEC_TIMEOUT_SECONDS` | `300` | [30, 7200] | Experiment execution wall-clock timeout | Set 300 for quick prototypes, 900 for medium, 3600 for GPU training |
| `MAX_EXEC_ATTEMPTS` | `3` | [1, 10] | Auto execution repair attempts | Increase if execution failures are usually fixable by LLM |
| `MAX_RESULT_REPAIR_ATTEMPTS` | `2` | [1, 5] | Auto result serialization repair attempts | Rarely need more than 2 |
| `REDUCED_DATASET_SIZE` | `1000` | [100, 100000] | Sample count used when recovering from MemoryError | Set to minimum viable for the algorithm (e.g., 500 for neural nets) |
| `MAX_EXPERIMENT_MINUTES` | `5` | [1, 120] | Told to PlannerAgent as design constraint | Set to actual available time budget |
| `MIN_DATASET_SIZE` | `100` | [10, 10000] | Below this, don't reduce further — escalate | Depends on algorithm minimum viable data size |
| `MIN_BATCH_SIZE` | `1` | [1, 16] | Below this, GPU OOM cannot be fixed by reducing batch | 1 for most models; 4 if model requires multiple samples |

---

## 6. Phase 4: Paper Writing

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `MAX_SECTION_REVISIONS` | `3` | [1, 10] | Max revisions per section before NEEDS_REVIEW | Increase for higher quality requirements; decrease for speed |
| `QUALITY_THRESHOLD` | `0.75` | [0.0, 1.0] | Paper-level quality score below which targeted rewrites occur | 0.75 = 3/4 checks pass; lower for lenient output, raise for strict |
| `SECTION_REWRITE_THRESHOLD` | `0.60` | [0.0, 1.0] | Section score below which it's added to rewrite list | Must be ≤ QUALITY_THRESHOLD |
| `MIN_WORDS_ABSTRACT` | `150` | [50, 500] | Abstract minimum word count | Academic standard: 150–250 |
| `MIN_WORDS_INTRODUCTION` | `400` | [100, 2000] | Introduction minimum | Increase for longer papers |
| `MIN_WORDS_RELATED_WORKS` | `400` | [100, 2000] | Related Works minimum | Decrease for workshop papers |
| `MIN_WORDS_PROPOSED_METHOD` | `500` | [200, 3000] | Method section minimum | Increase if method is complex |
| `MIN_WORDS_EXPERIMENTS` | `600` | [300, 3000] | Experiments section minimum | Decrease for ablation-only reports |
| `MIN_WORDS_CONCLUSION` | `200` | [50, 1000] | Conclusion minimum | 200 is standard |
| `MIN_REFERENCE_ENTRIES` | `5` | [3, 50] | Minimum bibliography entries | Increase for full conference papers |
| `CONTEXT_BANK_MAX_CHARS` | `3000` | [500, 10000] | Prior section context passed to WriterAgent | Increase for better coherence (at higher token cost) |

---

## 7. LLM and API

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `LLM_MODEL` | `claude-sonnet-4-6` | any valid model | Which Claude model to use | Change to Haiku for speed, Opus for quality |
| `MAX_LLM_RETRIES` | `10` | [3, 20] | Retries for transient LLM errors | Lower if using reliable API tier; raise for shared/rate-limited keys |
| `BACKOFF_BASE_SEC` | `0.5` | [0.1, 5.0] | Base exponential backoff delay | Lower for low-latency requirements; raise for heavily throttled APIs |
| `BACKOFF_MAX_SEC` | `32.0` | [5.0, 120.0] | Maximum backoff delay | Raise if hitting persistent 529s on busy deployments |
| `BACKOFF_JITTER_FRACTION` | `0.25` | [0.0, 0.5] | Random jitter added to backoff | 0.25 prevents thundering herd in multi-run scenarios |
| `MAX_TOKENS_CODER` | `8192` | [1024, 32768] | Token budget for CoderAgent responses | 8192 is needed for files >200 lines; reduce if cost is a concern |
| `TEMPERATURE_EXECUTOR` | `0.0` | [0.0, 0.3] | ExecutorAgent temperature | Always 0.0 — deterministic reporting is critical |
| `TEMPERATURE_CODER` | `0.2` | [0.0, 0.7] | CoderAgent temperature | 0.0 for pure repair; 0.3–0.5 if initial writes are too repetitive |
| `TEMPERATURE_WRITER` | `0.4` | [0.1, 0.8] | WriterAgent temperature | 0.4 balances variety and coherence; raise for more creative prose |

---

## 8. Tool Result Budgets

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `MAX_TOOL_RESULT_CHARS` | `4000` | [500, 20000] | Per-tool result truncation limit | Increase if agents lose important context from long stdout; decrease if hitting context limits |
| `MAX_OUTPUT_TAIL_CHARS` | `2000` | [200, 10000] | stdout/stderr tail in ExecutorResult | Increase for verbose experiment outputs |
| `MAX_STDERR_DISPLAY_LINES` | `50` | [10, 200] | Lines of stderr shown to user in escalation | 50 is enough for most stack traces |

Truncation format:
```python
def apply_tool_result_budget(result: str) -> str:
    if len(result) <= MAX_TOOL_RESULT_CHARS:
        return result
    half = MAX_TOOL_RESULT_CHARS // 2
    trimmed = len(result) - MAX_TOOL_RESULT_CHARS
    return (
        result[:half]
        + f"\n...[TRUNCATED {trimmed} chars — increase MAX_TOOL_RESULT_CHARS to see more]...\n"
        + result[-half:]
    )
```

---

## 9. Token Budgets

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `CONTEXT_NEAR_LIMIT_FRACTION` | `0.90` | [0.70, 0.95] | Fraction of context window at which a warning is logged | Lower if agents start making errors near context limit; raise for long experiments |
| `DIMINISHING_RETURNS_THRESHOLD` | `500` | [100, 2000] | Output tokens below which a turn is "low delta" | Lower for agents that naturally produce short turns; raise for verbose agents |
| `DIMINISHING_RETURNS_WINDOW` | `3` | [2, 6] | Consecutive low-delta turns before declaring stuck | 3 is a good default; raise if legitimate coding involves many short turns |

Diminishing returns logic:
```python
class TokenBudgetMonitor:
    def __init__(self):
        self.deltas: list[int] = []

    def record(self, output_tokens: int, input_tokens: int, context_window: int) -> str:
        self.deltas.append(output_tokens)

        if (
            len(self.deltas) >= DIMINISHING_RETURNS_WINDOW
            and all(d < DIMINISHING_RETURNS_THRESHOLD for d in self.deltas[-DIMINISHING_RETURNS_WINDOW:])
        ):
            return "STUCK"

        if input_tokens / context_window > CONTEXT_NEAR_LIMIT_FRACTION:
            return "NEAR_LIMIT"

        return "OK"
```

---

## 10. User Interaction

| Constant | Default | Range | Effect | When to Tune |
|----------|---------|-------|--------|--------------|
| `ESCALATION_RATE_LIMIT_SEC` | `5.0` | [0, 60] | Min seconds between escalation displays | Set to 0 for automated testing; raise if rapid-fire escalations are disorienting |
| `ESCALATION_HISTORY_DISPLAY` | `5` | [1, 20] | Prior attempts shown in escalation message | Raise to 10 for complex debugging sessions; lower for concise output |
| `ESCALATION_TIMEOUT_SECONDS` | `0` | [0, 86400] | Escalation response timeout; 0 = block forever | Set to 300 for unattended batch runs with fallback to "skip"; 0 for interactive |

---

## 11. Tuning Guide by Scenario

### Scenario A: Fast Prototype (speed over quality)

```python
# config.yaml
exec_timeout_seconds:      120     # 2 min
max_auto_attempts:         2       # fail fast → human
designer_max_files:        6       # keep it small
max_section_revisions:     1       # first draft is good enough
quality_threshold:         0.50    # lenient
min_words_experiments:     300     # shorter reports
approval_timeout_seconds:  60      # user must respond quickly
```

### Scenario B: Overnight Batch Run (unattended)

```python
# config.yaml
exec_timeout_seconds:      3600    # 1 hour for training
max_auto_attempts:         10      # try hard before giving up
approval_loop_max:         1       # auto-approve (or set timeout + auto-approve)
approval_timeout_seconds:  30      # short timeout → auto-approve on timeout
escalation_timeout_seconds: 30     # short escalation timeout → auto-skip
max_section_revisions:     5
quality_threshold:         0.70
```

### Scenario C: High-Quality Research Paper

```python
# config.yaml
exec_timeout_seconds:      7200    # 2 hours
max_auto_attempts:         5       # default
approval_loop_max:         20      # user may refine plan many times
approval_timeout_seconds:  0       # block forever (user is present)
max_section_revisions:     5
quality_threshold:         0.85
section_rewrite_threshold: 0.70
min_words_introduction:    600
min_words_experiments:     1000
min_words_proposed_method: 800
min_reference_entries:     15
context_bank_max_chars:    6000
temperature_writer:        0.5
```

### Scenario D: Debugging a Failing Pipeline

```python
# config.yaml
stuck_loop_threshold:      2       # escalate sooner
circular_import_escalate_after: 1  # circular imports → human immediately
max_tool_result_chars:     10000   # see full tool output
max_output_tail_chars:     5000    # see more stderr
escalation_history_display: 10     # show all repair history
```

### Scenario E: GPU Cluster Run

```python
# config.yaml
exec_timeout_seconds:      10800   # 3 hours
min_batch_size:            4       # GPU requires at least 4 samples
reduced_dataset_size:      5000    # can handle more data even reduced
max_experiment_minutes:    60      # tell planner to plan accordingly
```

---

## Constant Dependency Diagram

Some constants have dependencies:

```
QUALITY_THRESHOLD >= SECTION_REWRITE_THRESHOLD
  (if QUALITY_THRESHOLD < SECTION_REWRITE_THRESHOLD,
   sections that need rewriting won't be flagged for the overall score)

MAX_AUTO_ATTEMPTS + DEFAULT_EXTRA_ATTEMPTS
  = Total attempts before requiring a second user interaction
  (user chooses C once → max_auto increases by DEFAULT_EXTRA_ATTEMPTS)

CODER_MAX_TURNS_WRITE >= DESIGNER_MAX_FILES × 3
  (each file needs at least: write turn, syntax_check turn, import_check turn)
  Minimum: DESIGNER_MAX_FILES=12 → CODER_MAX_TURNS_WRITE should be ≥ 36

CONTEXT_BANK_MAX_CHARS × len(PAPER_WRITE_ORDER)
  = Approximate additional tokens per section for coherence context
  Default: 3000 × 7 = 21000 chars ≈ 5000 tokens across all sections
```
