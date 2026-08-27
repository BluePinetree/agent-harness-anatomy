# CrewAI 안정화 구현 계획서

> **작성**: Alex (Codex 설계 엔지니어) × Jordan (Claude Code 설계 엔지니어)  
> **근거**: `expert_review_crewai_stability_ko.md` 3회 검토 세션 종합  
> **작성일**: 2026-05-27  
> **목표**: 연구 주제 입력 → 실험 실행 → 논문 완성까지, 중단 없이 안정적으로 완료

---

## 1. 목표 선언문

> **"연구자가 주제를 입력하면, 시스템이 알아서 실험을 설계·구현·실행·분석하고  
> 논문 초안을 완성한다. 에러로 인한 중단은 없고, 환경 문제는 즉시 알려주며,  
> 실험 중에도 방향을 조정할 수 있다."**

### 성공 기준 (Definition of Done)

| 항목 | 기준 |
|------|------|
| **안정성** | CIFAR-100 분류 실험 5회 연속 논문 완성 (중간 수동 개입 없음) |
| **복구성** | 어느 Phase든 강제 종료 후 재시작 시 해당 Phase부터 재개 |
| **투명성** | Phase 3 실행 중 epoch 진행률 및 loss가 실시간으로 표시됨 |
| **대화성** | 실행 전 3~4개 질문으로 실험 설정 확정, 실행 후 후속 실험 1클릭 시작 |
| **진단성** | 실패 시 "왜 실패했는가"를 run_summary.json으로 확인 가능 |

---

## 2. 현재 상태 진단 (As-Is)

### 2.1 파이프라인 구조

```
Phase 0: Workspace 생성
Phase 1: Planner (CrewAI) → Designer (CrewAI) → PlanBundle
Phase 2: Coder (직접 LLM 호출) → repair loop → CodingResult  ← 2026-05-21 개선
Phase 3: Executor (subprocess) → AnalyzerAgent (CrewAI) → ExecutorResult
Phase 4: Writer (CrewAI 섹션별) → WritingResult
```

### 2.2 확인된 취약점 목록

#### 그룹 A — 실행 중단 시 복구 불가
| ID | 취약점 | 발생 조건 | 현재 결과 |
|----|--------|---------|---------|
| A1 | Phase 간 체크포인트 없음 | 어느 Phase든 종료 시 | Phase 0부터 전체 재실행 |
| A2 | 파일 단위 체크포인트 없음 | Phase 2 중간 종료 시 | 생성된 파일도 재생성 |

#### 그룹 B — 오류 처리 비효율
| ID | 취약점 | 발생 조건 | 현재 결과 |
|----|--------|---------|---------|
| B1 | 오류 분류 없음 | `pip install` 필요한 환경 오류 | repair 3회 시도 후 GuidanceGate |
| B2 | repair 전략 단조로움 | 동일 프롬프트 반복 | 같은 실패 패턴 반복 |
| B3 | 의존성 무효화 없음 | 파일 A repair 후 | A를 import하는 B가 stale |
| B4 | 컨텍스트 잘림 무음 | `_MAX_DEP_CHARS` 초과 | LLM이 잘린 줄 모름 |

#### 그룹 C — 컨텍스트 관리 부재
| ID | 취약점 | 발생 조건 | 현재 결과 |
|----|--------|---------|---------|
| C1 | 4계층 메모리 미분리 | repair 반복 시 | 프롬프트에 오래된 이력 누적 |
| C2 | Phase 경계 압축 없음 | Phase 4 Writer 진입 시 | raw 로그가 논문 프롬프트에 혼입 |
| C3 | 토큰 소비 미추적 | Phase 2 장기 실행 시 | 예고 없이 토큰 한도 도달 |

#### 그룹 D — 관찰가능성 부재
| ID | 취약점 | 발생 조건 | 현재 결과 |
|----|--------|---------|---------|
| D1 | Phase 3 블로킹 실행 | 30분 학습 중 | 사용자에게 아무 피드백 없음 |
| D2 | repair 이력 소멸 | Phase 완료 시 | 반복 패턴 분석 불가 |
| D3 | 의미 오류 미검출 | num_classes=10 (CIFAR-100) | smoke test 통과, Phase 3에서 발견 |

#### 그룹 E — 사용자 개입 구조 부재
| ID | 취약점 | 발생 조건 | 현재 결과 |
|----|--------|---------|---------|
| E1 | 실험 전 설정 확인 없음 | 모호한 연구 주제 입력 | 잘못된 방향으로 30분 실행 |
| E2 | 실행 중 방향 주입 불가 | 진행 중 인사이트 발생 시 | 새 run 전체 재시작 필요 |
| E3 | 후속 실험 제안 없음 | Phase 4 완료 후 | 단발성 결과로 끝남 |

---

## 3. 목표 아키텍처 (To-Be)

### 3.1 전체 구조

```
┌─────────────────────────────────────────────────────┐
│  레이어 1: Pre-flight Clarification                  │
│  (MAX 4개 질문, default 있음, 60초 타임아웃)          │
└───────────────────┬─────────────────────────────────┘
                    │ EnrichedTopic
                    ▼
┌─────────────────────────────────────────────────────┐
│  핵심 배치 파이프라인 (결정론적 실행)                  │
│                                                      │
│  Phase 0: WorkspaceManager                          │
│      ↓ [checkpoint: phase0_workspace.json]          │
│  Phase 1: Planner (직접 LLM) + Designer (직접 LLM)  │
│      ↓ [checkpoint: phase1_plan_bundle.json]        │
│      ↓ [ApprovalGate + InjectionQueue 소비]         │
│  Phase 2: Coder (직접 LLM)                          │
│      ├─ PipelineMemory (4계층)                      │
│      ├─ RepairStrategyRotator                       │
│      ├─ ErrorClassifier                             │
│      ├─ DependencyInvalidationGraph                 │
│      ├─ TokenBudgetTracker                          │
│      └─ [checkpoint: phase2_file_<hash>.done x N]  │
│      ↓ [checkpoint: phase2_coding_result.json]      │
│      ↓ [SemanticValidator: dry-run 1 epoch]         │
│  Phase 3: Executor (async streaming)                │
│      ├─ FailurePatternDetector                      │
│      ├─ PlanAdaptationGate (반복 실패 시)            │
│      └─ ContextCompressor (→ Phase 4 입력)          │
│      ↓ [checkpoint: phase3_exec_result.json]        │
│  Phase 4: Writer (직접 LLM 섹션별)                  │
│      └─ ContextCompressor (Phase 1+2+3 압축 입력)   │
│      ↓ [checkpoint: phase4_writing_result.json]     │
│  run_summary.json 생성                              │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  레이어 3: Extension Proposer                        │
│  (후속 실험 2~3개 제안, ExtensionRouter로 최소 재실행)│
└─────────────────────────────────────────────────────┘
```

### 3.2 신규 컴포넌트 목록

| 컴포넌트 | 위치 | 역할 |
|---------|------|------|
| `CheckpointManager` | `core/checkpoint.py` | Phase/파일 단위 상태 영속화 및 복원 |
| `ErrorClassifier` | `core/error_classifier.py` | 오류를 5종으로 분류, repair 전략 결정 |
| `RepairStrategyRotator` | `core/repair_strategy.py` | targeted→fresh→stub 순환 |
| `DependencyInvalidationGraph` | `core/dep_graph.py` | 파일 수정 후 의존 파일 재검사 큐 |
| `PipelineMemory` | `core/pipeline_memory.py` | working/episodic/semantic/archival 4계층 |
| `ContextCompressor` | `core/context_compressor.py` | Phase 경계 데이터 압축 |
| `TokenBudgetTracker` | `core/token_budget.py` | Phase 2 LLM 호출 토큰 누적 추적 |
| `FailurePatternDetector` | `core/failure_patterns.py` | OOM/timeout 반복 패턴 감지 |
| `PlanAdaptationGate` | `orchestration/adaptation_gate.py` | 실행 불가 환경 감지 → 계획 수정 제안 |
| `SemanticValidator` | `core/semantic_validator.py` | dry-run으로 API contract 검증 |
| `PreflightClarifier` | `core/preflight.py` | 실행 전 최소 질문 생성 및 응답 반영 |
| `ContextInjectionQueue` | `orchestration/injection_queue.py` | 실행 중 사용자 주입 큐 |
| `ExtensionProposer` | `core/extension_proposer.py` | Phase 4 완료 후 후속 실험 제안 |
| `ExtensionRouter` | `orchestration/extension_router.py` | 최소 재실행 지점 결정 |

---

## 4. 구현 로드맵

전체 구현을 4개 Sprint로 나눈다. 각 Sprint는 독립적으로 배포 및 검증 가능하다.

---

### Sprint 1: 실행 안정성 기반 (1~2주)

**목표**: "어디서 실패해도 거기서 재시작한다"

#### 작업 1-1: `CheckpointManager` 구현

**파일**: `crewai_prototype/core/checkpoint.py`

```python
class CheckpointManager:
    def __init__(self, run_id: str, workspace_root: str):
        self.base = Path(workspace_root) / "checkpoints"
        self.base.mkdir(parents=True, exist_ok=True)
    
    # Phase 단위 저장/복원
    def save_phase(self, phase_name: str, result: BaseModel) -> None:
        path = self.base / f"{phase_name}.json"
        path.write_text(result.model_dump_json(indent=2), encoding="utf-8")
    
    def load_phase(self, phase_name: str, model_cls: type[T]) -> T | None:
        path = self.base / f"{phase_name}.json"
        if path.exists():
            return model_cls.model_validate_json(path.read_text())
        return None
    
    # 파일 단위 저장/확인 (Phase 2용)
    def mark_file_done(self, relative_path: str) -> None:
        key = hashlib.md5(relative_path.encode()).hexdigest()[:8]
        (self.base / f"file_{key}.done").touch()
    
    def is_file_done(self, relative_path: str) -> bool:
        key = hashlib.md5(relative_path.encode()).hexdigest()[:8]
        return (self.base / f"file_{key}.done").exists()
```

**통합 지점**:
- `phases/phase2_coding.py`: `_repair_loop()` 완료 시 `checkpoint.mark_file_done()`
- `phases/phase3_execution.py`: `run_execution_phase()` 완료 시 `checkpoint.save_phase("phase3_exec")`
- `main.py`: 시작 시 각 Phase 체크포인트 존재 여부 확인, 있으면 해당 Phase 스킵

**검증**: Phase 2 10번째 파일 생성 직후 프로세스 강제 종료 → 재시작 시 11번째 파일부터 시작 확인

---

#### 작업 1-2: `ErrorClassifier` 구현

**파일**: `crewai_prototype/core/error_classifier.py`

```python
class ErrorClass(Enum):
    TRANSIENT = "transient"           # API 오버로드, 네트워크 → 재시도
    RECOVERABLE_CODE = "code"         # 구문/import 오류 → LLM repair
    RECOVERABLE_LOGIC = "logic"       # 런타임 로직 → fresh_from_spec
    FATAL_ENV = "fatal_env"           # 환경 설정 → 즉시 사용자 알림
    FATAL_SEMANTIC = "fatal_semantic" # 의미 오류 → GuidanceGate

# 분류 규칙 (LLM 호출 없이 패턴 매칭)
_FATAL_ENV_PATTERNS = [
    r"No module named '(torch|torchvision|numpy|pandas|sklearn)'",
    r"CUDA out of memory",
    r"CUDA driver version is insufficient",
    r"No GPU available",
    r"Permission denied",
]

def classify(check: CheckResult) -> ErrorClass:
    for pattern in _FATAL_ENV_PATTERNS:
        if re.search(pattern, check.error):
            return ErrorClass.FATAL_ENV
    
    if check.error_type == "syntax":
        return ErrorClass.RECOVERABLE_CODE
    if check.error_type == "import":
        return ErrorClass.RECOVERABLE_CODE
    if check.error_type == "runtime":
        if "RecursionError" in check.error or "MemoryError" in check.error:
            return ErrorClass.FATAL_ENV
        return ErrorClass.RECOVERABLE_LOGIC
    return ErrorClass.RECOVERABLE_CODE
```

**통합 지점**:
- `phases/phase2_coding.py`의 `_repair_loop()`: repair 시도 전 `classify(check)` 호출
  - `FATAL_ENV` → repair 루프 건너뜀, 즉시 `emit("FATAL_ENV_ERROR", {reason})`
  - `FATAL_SEMANTIC` → GuidanceGate (즉시)
  - 그 외 → 기존 repair 루프 진행

**검증**: `No module named 'torch'` 오류 발생 시 repair 시도 없이 즉시 환경 오류 메시지 출력

---

#### 작업 1-3: 컨텍스트 잘림 명시

**파일**: `crewai_prototype/phases/phase2_coding.py`  
**변경 위치**: `_build_dep_context()` 함수

```python
# 기존 (조용히 자름)
content = file_content[:_MAX_DEP_CHARS]

# 변경 (잘림 명시)
if len(file_content) > _MAX_DEP_CHARS:
    content = file_content[:_MAX_DEP_CHARS]
    content += (
        f"\n\n# ──── TRUNCATED ({len(file_content)} chars → {_MAX_DEP_CHARS}) ────\n"
        f"# Only the above portion was provided. Rely solely on the exported\n"
        f"# names/signatures visible above; do not assume anything beyond this."
    )
else:
    content = file_content
```

**검증**: 6,000자 초과 파일의 의존성 컨텍스트에 TRUNCATED 마커 확인

---

#### Sprint 1 완료 기준

- [ ] 체크포인트 파일이 `outputs/<run_id>/checkpoints/`에 생성됨
- [ ] Phase 2 도중 강제 종료 후 재시작 시 완료된 파일 재생성하지 않음
- [ ] `No module named 'torch'` 에러 → repair 시도 없이 환경 오류 즉시 표시
- [ ] `_MAX_DEP_CHARS` 초과 시 TRUNCATED 마커 프롬프트에 포함됨

---

### Sprint 2: Repair 내성 강화 (1~2주)

**목표**: "같은 에러에서 빠져나온다. 컨텍스트가 오염되지 않는다."

#### 작업 2-1: `RepairStrategyRotator` 구현

**파일**: `crewai_prototype/core/repair_strategy.py`

```python
class RepairStrategy(Enum):
    TARGETED_FIX = "targeted_fix"       # 에러 라인만, 기존 코드 유지
    FRESH_FROM_SPEC = "fresh_from_spec" # 이전 코드 제외, spec에서 새로 생성
    MINIMAL_STUB = "minimal_stub"       # 최소 구현 + TODO, 파이프라인 진행
    USER_ESCALATION = "user_escalation" # GuidanceGate

def select_strategy(
    attempt: int,
    error_class: ErrorClass,
    consecutive_same_error: int,
) -> RepairStrategy:
    # 같은 에러가 2번 이상 반복되면 전략 상향
    effective_attempt = attempt + (1 if consecutive_same_error >= 2 else 0)
    
    if effective_attempt == 1:
        return RepairStrategy.TARGETED_FIX
    elif effective_attempt == 2:
        return RepairStrategy.FRESH_FROM_SPEC
    elif effective_attempt == 3:
        return RepairStrategy.MINIMAL_STUB
    else:
        return RepairStrategy.USER_ESCALATION
```

**`_repair_content()` 수정**:
- `TARGETED_FIX`: 기존 방식 (이전 코드 포함)
- `FRESH_FROM_SPEC`: `previous_content` 완전 제외, spec만 전달
- `MINIMAL_STUB`: LLM 호출 없이 즉시 stub 작성

**검증**: 동일 syntax 오류 3회 반복 → 3번째 시도는 `FRESH_FROM_SPEC` 전략 사용 확인

---

#### 작업 2-2: `PipelineMemory` 4계층 구조

**파일**: `crewai_prototype/core/pipeline_memory.py`

```python
@dataclass
class SemanticContext:
    """변하지 않는 도메인 지식 — Phase 2 전체에서 동일"""
    stack_rule: str
    num_classes: int
    dataset_name: str
    entry_point: str
    key_exports: dict[str, list[str]]  # 파일명 → 공개 API 목록

@dataclass
class WorkingContext:
    """현재 파일 생성에만 필요한 것"""
    file_spec: FileNodeSpec
    dep_context: str    # 이미 압축/잘린 의존성 컨텍스트
    current_error: str  # 최신 에러만 (이전 것 제외)
    strategy: RepairStrategy

@dataclass
class EpisodicSummary:
    """현재 Phase 내 진행 요약 — 프롬프트에 한 줄만"""
    files_done: int
    files_total: int
    recent_errors: list[str]  # 최근 3개만

class PipelineMemory:
    semantic: SemanticContext
    episodic_summary: EpisodicSummary  # 프롬프트에 포함
    episodic_full: list[FileResult]    # 디스크에만 (체크포인트)
    archival_paths: list[str]          # 로그 파일 경로만

    def build_generation_prompt(self, ctx: WorkingContext) -> str:
        """LLM에 전달할 프롬프트 — 필요한 계층만 조합"""
        return "\n\n".join(filter(None, [
            self.semantic.to_block(),
            ctx.file_spec.to_block(),
            ctx.dep_context,
            f"Pipeline: {self.episodic_summary.to_line()}",
        ]))
    
    def build_repair_prompt(self, ctx: WorkingContext) -> str:
        """repair 프롬프트 — 전략에 따라 이전 코드 포함/제외"""
        include_previous = ctx.strategy == RepairStrategy.TARGETED_FIX
        return "\n\n".join(filter(None, [
            self.semantic.to_block(),
            ctx.file_spec.to_block(),
            ctx.dep_context,
            f"Error: {ctx.current_error}",  # 최신 에러만
            f"Previous code:\n{ctx.previous_code}" if include_previous else
            "# Start fresh from spec — do not reference any previous attempt.",
        ]))
```

**검증**: repair 3회 반복 후 프롬프트 크기가 1회 반복보다 크게 증가하지 않음

---

#### 작업 2-3: `DependencyInvalidationGraph` 구현

**파일**: `crewai_prototype/core/dep_graph.py`

```python
class DependencyInvalidationGraph:
    def __init__(self, file_specs: list[FileNodeSpec]):
        self._dependents: dict[str, set[str]] = {}
        for spec in file_specs:
            for dep in spec.imports_from:
                self._dependents.setdefault(dep, set()).add(spec.path)
    
    def invalidated_by(self, modified_path: str) -> set[str]:
        """수정된 파일에 의존하는 모든 파일 (전이적 포함)"""
        visited, queue = set(), [modified_path]
        while queue:
            cur = queue.pop()
            for dep in self._dependents.get(cur, set()):
                if dep not in visited:
                    visited.add(dep)
                    queue.append(dep)
        return visited
```

**통합 지점** (`phases/phase2_coding.py`):
```python
# repair로 파일이 변경됐을 때
if file_was_repaired:
    stale_files = dep_graph.invalidated_by(file_result.path)
    for stale_path in stale_files:
        if not already_checked(stale_path):
            recheck_queue.add(stale_path)
```

**검증**: `config.py`가 repair 후 `config.py`를 import하는 `trainer.py`의 check 상태가 `needs_recheck`로 표시됨

---

#### 작업 2-4: `ContextCompressor` — Phase 경계 압축

**파일**: `crewai_prototype/core/context_compressor.py`

```python
class ContextCompressor:
    
    @staticmethod
    def coding_to_handoff(result: CodingResult) -> CodingHandoffSummary:
        """Phase 2 → 3 핸드오프: 핵심 사실만"""
        return CodingHandoffSummary(
            files_written=sum(1 for f in result.all_files if f.written),
            files_stubbed=sum(1 for f in result.all_files if not f.written),
            stub_paths=[f.path for f in result.all_files if not f.written],
            notable_repairs=[
                f"{f.path}: {f.check.error_type} resolved in {len(f.repair_records)} attempts"
                for f in result.all_files if f.repair_records
            ],
            workspace_root=result.workspace_root,
        )
    
    @staticmethod
    def exec_to_writer_context(
        plan: PlannerResult,
        coding: CodingHandoffSummary,
        exec_result: ExecutorResult,
    ) -> WriterContext:
        """Phase 3 → 4 핸드오프: Writer가 논문 쓰기에 필요한 것만"""
        return WriterContext(
            # 연구 목적과 가설 (설계 세부사항 제외)
            objective=plan.objective,
            hypotheses=plan.hypotheses,
            
            # 아키텍처 요약 (repair 이력 제외)
            architecture_summary=(
                f"{coding.files_written}개 파일 구현. "
                f"{'스텁 ' + str(coding.files_stubbed) + '개 포함. ' if coding.files_stubbed else ''}"
            ),
            
            # 실험 결과 수치만 (전체 stdout 제외)
            metrics=exec_result.metrics,
            epoch_curve=exec_result.epoch_summary,  # 요약된 학습 곡선
            
            # 원시 데이터 경로 (Writer가 필요 시 직접 읽음)
            result_json_path=exec_result.result_json_path,
        )
```

**검증**: Phase 4 Writer 프롬프트에 raw stdout이 포함되지 않음, `WriterContext`의 필드만 포함됨

---

#### Sprint 2 완료 기준

- [ ] 동일 에러 3회 반복 → 3번째 시도는 다른 전략 (FRESH_FROM_SPEC) 사용
- [ ] repair 반복 시 프롬프트에 최신 에러 1개만 포함 (이전 에러 누적 없음)
- [ ] 파일 A repair 후 A에 의존하는 파일 B가 재검사 큐에 추가됨
- [ ] Phase 4 Writer 프롬프트 크기 < 기존의 40% (압축 효과)

---

### Sprint 3: 관찰가능성 강화 (1주)

**목표**: "무슨 일이 일어나는지 항상 보인다. 실패 이력이 남는다."

#### 작업 3-1: Phase 3 실시간 스트리밍

**파일**: `crewai_prototype/phases/phase3_execution.py`  
**변경**: `subprocess.run()` → `asyncio.create_subprocess_exec()` + 실시간 emit

```python
async def _run_script_streaming(
    entry_point: str,
    workspace_root: str,
    timeout_secs: int,
) -> AsyncIterator[ExecLine]:
    
    process = await asyncio.create_subprocess_exec(
        "python", entry_point,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        cwd=workspace_root,
    )
    
    metric_parser = TrainingMetricParser()
    
    async for raw_line in process.stdout:
        line = raw_line.decode("utf-8", errors="replace").rstrip()
        metric = metric_parser.parse(line)
        
        emit("EXEC_STDOUT", {
            "line": line,
            "metric": metric.dict() if metric else None,
            "ts": time.time(),
        })
        yield ExecLine(text=line, metric=metric)
    
    await asyncio.wait_for(process.wait(), timeout=timeout_secs)
    return process.returncode

class TrainingMetricParser:
    _PATTERN = re.compile(
        r"[Ee]poch\s*\[?(\d+)[/|](\d+)\]?"
        r".*?[Ll]oss[:\s]+([\d.]+)"
        r"(?:.*?[Aa]cc(?:uracy)?[:\s]+([\d.]+))?"
    )
    
    def parse(self, line: str) -> TrainingMetric | None:
        m = self._PATTERN.search(line)
        if not m:
            return None
        return TrainingMetric(
            epoch=int(m.group(1)),
            total_epochs=int(m.group(2)),
            loss=float(m.group(3)),
            accuracy=float(m.group(4)) if m.group(4) else None,
        )
```

**검증**: Phase 3 실행 중 UI에서 `Epoch [5/50], Loss: 1.234` 형태의 실시간 업데이트 확인

---

#### 작업 3-2: `TokenBudgetTracker` 구현

**파일**: `crewai_prototype/core/token_budget.py`

```python
@dataclass
class TokenBudgetTracker:
    budget_input: int = 500_000   # Phase 2 전체 입력 토큰 한도
    total_input: int = 0
    total_output: int = 0
    calls: int = 0
    
    WARN_PCT = 0.70
    PRESSURE_PCT = 0.85
    
    def record(self, usage: TokenUsage) -> BudgetStatus:
        self.total_input += usage.input_tokens
        self.total_output += usage.output_tokens
        self.calls += 1
        
        pct = self.total_input / self.budget_input
        
        if pct >= self.PRESSURE_PCT:
            emit("TOKEN_PRESSURE", {"pct_used": round(pct * 100)})
            return BudgetStatus.PRESSURE  # 이후 dep_context 크기 반감
        if pct >= self.WARN_PCT:
            emit("TOKEN_WARNING", {"pct_used": round(pct * 100)})
            return BudgetStatus.WARN
        return BudgetStatus.OK
```

**통합**: Phase 2에서 각 LLM 호출 후 `tracker.record(response.usage)` 호출  
`PRESSURE` 상태 → `_MAX_DEP_CHARS`를 절반으로 줄임

**검증**: 토큰 85% 도달 시 이후 프롬프트의 dep_context 크기가 3,000자로 감소

---

#### 작업 3-3: `run_summary.json` 생성

**파일**: `crewai_prototype/core/run_summary.py`

```python
def write_run_summary(
    run_id: str,
    workspace_root: str,
    topic: str,
    plan: PlannerResult | None,
    coding: CodingResult | None,
    exec_result: ExecutorResult | None,
    writing: WritingResult | None,
    duration_sec: float,
) -> Path:
    
    summary = {
        "run_id": run_id,
        "topic": topic,
        "completed_at": datetime.utcnow().isoformat(),
        "duration_sec": round(duration_sec),
        "phases": {
            "phase1": _summarize_phase1(plan),
            "phase2": _summarize_phase2(coding),
            "phase3": _summarize_phase3(exec_result),
            "phase4": _summarize_phase4(writing),
        },
        "overall_success": writing is not None and writing.complete,
    }
    
    path = Path(workspace_root) / "run_summary.json"
    path.write_text(json.dumps(summary, indent=2, ensure_ascii=False))
    return path

def _summarize_phase2(coding: CodingResult | None) -> dict:
    if not coding:
        return {"status": "skipped_or_failed"}
    
    all_files = coding.all_files
    repair_breakdown: dict[str, int] = {}
    for f in all_files:
        for r in f.repair_records:
            key = r.error_type or "unknown"
            repair_breakdown[key] = repair_breakdown.get(key, 0) + 1
    
    return {
        "files_generated": sum(1 for f in all_files if f.written),
        "files_stubbed": sum(1 for f in all_files if not f.written),
        "total_repairs": sum(len(f.repair_records) for f in all_files),
        "repair_breakdown": repair_breakdown,
        "smoke_test_passed": coding.smoke_test_passed,
    }
```

**검증**: 매 run 완료 후 `outputs/<run_id>/run_summary.json`이 생성됨

---

#### 작업 3-4: `SemanticValidator` — dry-run API contract 검사

**파일**: `crewai_prototype/core/semantic_validator.py`

```python
def validate_api_contract(
    workspace_root: str,
    plan: PlannerResult,
) -> CheckResult:
    """
    smoke test 후, 1 step dry-run으로 API contract 검증.
    - num_classes가 plan과 일치하는지
    - entry point 시그니처가 실행 가능한지
    """
    result = subprocess.run(
        ["python", plan.entry_point, "--dry-run", "--max-steps", "2"],
        capture_output=True,
        timeout=60,
        cwd=workspace_root,
        env={**os.environ, "DRY_RUN": "1"},
    )
    
    if result.returncode != 0:
        stderr = result.stderr.decode("utf-8", errors="replace")
        # num_classes 불일치 감지
        if f"num_classes" in stderr or "size mismatch" in stderr:
            return CheckResult(
                passed=False,
                error=stderr,
                error_type="semantic",
            )
        return CheckResult(passed=False, error=stderr, error_type="runtime")
    
    return CheckResult(passed=True)
```

**통합**: Phase 2 smoke test 통과 후, Phase 3 진입 전에 `validate_api_contract()` 실행  
실패 시 → Phase 2 repair loop 재진입 (semantic 오류로 분류)

**검증**: `num_classes=10`인 코드로 CIFAR-100 실험 시도 시 Phase 3 진입 전 오류 감지

---

#### Sprint 3 완료 기준

- [ ] Phase 3 실행 중 UI에 epoch/loss 실시간 표시
- [ ] Phase 2 완료 후 `run_summary.json`에 repair_breakdown 포함
- [ ] 토큰 85% 도달 시 TOKEN_PRESSURE 이벤트 발생
- [ ] num_classes 불일치 → Phase 3 진입 전 감지

---

### Sprint 4: 하이브리드 대화형 레이어 (1~2주)

**목표**: "실행 전에 방향을 맞추고, 결과를 보고 이어서 탐색한다."

#### 작업 4-1: `PreflightClarifier` — 최소 구조화 질문

**파일**: `crewai_prototype/core/preflight.py`

```python
class PreflightClarifier:
    MAX_QUESTIONS = 4
    TIMEOUT_SECS = 60
    
    def generate(self, raw_topic: str, capabilities: SystemCapabilities) -> list[ClarificationQuestion]:
        """LLM이 합리적으로 추론할 수 없는 것만 질문"""
        
        prompt = f"""
        Research topic: "{raw_topic}"
        
        Identify ONLY what cannot be reasonably defaulted:
        - Hardware constraints (GPU memory, time budget)
        - Evaluation priority (which metric matters most)
        - Comparison scope (which architectures/methods)
        - Allowed libraries (any restrictions)
        
        Rules:
        - Max {self.MAX_QUESTIONS} questions total
        - Every question MUST have a "default" value
        - Hyperparameters (lr, batch_size) = do NOT ask (use defaults)
        - Code structure = do NOT ask (Designer decides)
        
        Output: JSON array of questions with fields:
          dimension, question, options (2-4 items or null), default
        """
        
        raw = self.llm.call([{"role": "user", "content": prompt}])
        questions = parse_json(raw)
        
        # default 없는 질문 필터링 (항상 진행 가능해야 함)
        return [q for q in questions if q.get("default")][:self.MAX_QUESTIONS]
    
    def apply(self, topic: str, answers: dict, questions: list) -> EnrichedTopic:
        resolved = {
            q["dimension"]: answers.get(q["dimension"], q["default"])
            for q in questions
        }
        
        constraint_lines = [f"- {k}: {v}" for k, v in resolved.items()]
        enriched_prompt = (
            f"{topic}\n\nConstraints:\n" + "\n".join(constraint_lines)
        )
        
        return EnrichedTopic(
            original=topic,
            constraints=resolved,
            enriched_prompt=enriched_prompt,
        )
```

**UX 플로우** (`main.py`):
```
사용자 입력: "CIFAR-100 분류 성능 비교"

[PreflightClarifier 작동]
━━━━━━━━━━━━━━━━━━━━━━━━━━
실험 전 3가지를 확인할게요. 스킵 시 기본값으로 자동 진행합니다. (60초)

1. 비교 아키텍처: (기본: ResNet-18 vs ViT-Tiny)
   a) ResNet-18 vs ViT-Tiny  b) ResNet-50 vs ViT-Base  c) 직접 입력

2. 최우선 평가 지표: (기본: Top-1 Accuracy)
   a) Top-1 Accuracy  b) Accuracy + 속도  c) Accuracy + 파라미터 수

3. GPU 메모리 제약: (기본: 16GB)
   a) 8GB 이하  b) 16GB  c) 24GB 이상
━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**검증**: 60초 타임아웃 시 기본값으로 자동 진행 확인

---

#### 작업 4-2: `ContextInjectionQueue` — 실행 중 컨텍스트 주입

**파일**: `crewai_prototype/orchestration/injection_queue.py`

```python
@dataclass
class InjectionEntry:
    message: str
    injected_at_phase: str
    timestamp: float
    status: str = "pending"  # pending | consumed | discarded
    intent: InjectionIntent | None = None

class ContextInjectionQueue:
    _queue: list[InjectionEntry] = field(default_factory=list)
    
    # 주입 가능한 Phase (실행 중 코드 변경 불가 원칙)
    SAFE_INJECTION_PHASES = {"phase1_pending", "phase4_pending"}
    QUEUE_ONLY_PHASES = {"phase2_running", "phase3_running"}
    
    def inject(self, message: str, current_phase: str):
        entry = InjectionEntry(
            message=message,
            injected_at_phase=current_phase,
            timestamp=time.time(),
        )
        self._queue.append(entry)
        
        # Phase 2/3 실행 중이면 경고
        if current_phase in self.QUEUE_ONLY_PHASES:
            emit("INJECTION_QUEUED", {
                "message": "현재 실행 중. 다음 단계 시작 시 반영됩니다.",
                "will_apply_at": self._next_boundary(current_phase),
            })
        else:
            emit("INJECTION_ACCEPTED", {"will_apply_now": True})
    
    def consume(self, phase: str) -> list[InjectionEntry]:
        """Phase 시작 시 큐 소비"""
        pending = [e for e in self._queue if e.status == "pending"]
        for e in pending:
            e.status = "consumed"
            e.consumed_at_phase = phase
        return pending
```

**통합**: 각 Phase 시작 부분에서 `queue.consume(phase_name)` 호출  
소비된 항목 → `InjectionInterpreter`로 해석 → Phase 입력에 추가

**검증**: Phase 3 실행 중 "dropout 추가해봐" 주입 → Phase 4 시작 시 context에 포함됨

---

#### 작업 4-3: `FailurePatternDetector` + `PlanAdaptationGate`

**파일**: `crewai_prototype/core/failure_patterns.py`

```python
class FailurePattern(Enum):
    GPU_OOM = "gpu_oom"
    TIMEOUT = "timeout"
    IMPORT_MISSING = "import_missing"
    DIVERGING_LOSS = "diverging_loss"
    NONE = "none"

class FailurePatternDetector:
    
    def detect(self, exec_history: list[ExecAttempt]) -> FailurePattern:
        errors = [a.error_type for a in exec_history]
        metrics = [a.metrics for a in exec_history if a.metrics]
        
        if errors.count("CUDA out of memory") >= 2:
            return FailurePattern.GPU_OOM
        if errors.count("TIMEOUT") >= 2:
            return FailurePattern.TIMEOUT
        if errors.count("ModuleNotFoundError") >= 1:
            return FailurePattern.IMPORT_MISSING
        if len(metrics) >= 3 and self._is_diverging(metrics):
            return FailurePattern.DIVERGING_LOSS
        return FailurePattern.NONE
    
    def _is_diverging(self, metrics: list[dict]) -> bool:
        losses = [m.get("loss", 0) for m in metrics]
        return len(losses) >= 3 and losses[-1] > losses[0] * 2.0

class PlanAdaptationGate:
    
    _PROPOSALS = {
        FailurePattern.GPU_OOM: [
            "ViT-Tiny + ResNet-18 (소형)으로 축소 (예상 -60% GPU 사용)",
            "batch_size 128 → 32로 축소 후 재시도",
            "CIFAR-100 → CIFAR-10으로 데이터셋 축소",
        ],
        FailurePattern.TIMEOUT: [
            f"max_epoch를 {'{original}'} → {'{original//3}'}으로 축소",
            "검증 주기 늘리기 (val_every_n_epochs=10)",
        ],
        FailurePattern.DIVERGING_LOSS: [
            "학습률 10배 감소 (lr: 1e-3 → 1e-4)",
            "gradient clipping 추가 (max_norm=1.0)",
        ],
    }
    
    def propose(self, pattern: FailurePattern, plan: PlannerResult) -> AdaptationProposal | None:
        if pattern == FailurePattern.NONE:
            return None
        options = self._PROPOSALS.get(pattern, [])
        return AdaptationProposal(
            trigger=pattern.value,
            options=options,
            requires_phase2_restart=pattern in {
                FailurePattern.GPU_OOM, FailurePattern.IMPORT_MISSING
            },
        )
```

**검증**: OOM 2회 연속 → Phase 3 repair 루프 대신 적응 제안 UI 표시

---

#### 작업 4-4: `ExtensionProposer` — 후속 실험 제안

**파일**: `crewai_prototype/core/extension_proposer.py`

```python
class ExtensionProposer:
    MAX_PROPOSALS = 3
    
    def propose(
        self,
        plan: PlannerResult,
        exec_result: ExecutorResult,
        writing: WritingResult,
    ) -> list[ExtensionProposal]:
        
        prompt = f"""
        Completed: {plan.objective}
        Results: {json.dumps(exec_result.metrics, indent=2)}
        Key finding: {writing.conclusion_summary}
        
        Propose {self.MAX_PROPOSALS} concrete follow-up experiments.
        Each must:
        1. Address an open question from these results
        2. Be runnable with the same pipeline
        3. Estimate minutes to complete
        4. Specify which phases need to change (phase1/phase2/phase3/all)
        
        Sort by estimated time (shortest first).
        Output JSON array.
        """
        
        raw = self.llm.call([{"role": "user", "content": prompt}])
        proposals = parse_proposals(raw)
        
        # ExtensionRouter로 최소 재시작 지점 계산
        router = ExtensionRouter()
        for p in proposals:
            p.restart_from = router.determine(p, current_checkpoint)
        
        return proposals[:self.MAX_PROPOSALS]
```

**UX 플로우**:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
실험 완료. ResNet-18: 72.1%, ViT-Tiny: 69.4% (CIFAR-100)

후속 실험 제안 (선택 또는 Enter로 완료):

  [A] Data Augmentation 추가           +15분  Phase 2 일부 수정
      CutMix + AutoAugment → 예상 +2~3%p

  [B] 학습률 스케줄러 비교              +20분  Phase 2 일부 수정
      CosineAnnealing vs StepLR → ViT 성능 영향 분석

  [C] 더 큰 모델 비교                  +45분  Phase 1부터 재시작
      ResNet-50 vs ViT-Small → 성능 상한 파악

  [완료] 현재 결과로 마무리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
(60초 타임아웃 → 자동 완료)
```

**검증**: 옵션 A 선택 시 Phase 1 재실행 없이 Phase 2 일부 파일만 재생성 후 Phase 3 시작 확인

---

#### Sprint 4 완료 기준

- [ ] 실험 전 구조화 질문 1~4개 표시, 60초 타임아웃 후 기본값으로 자동 진행
- [ ] Phase 3 실행 중 주입 메시지 → 큐 저장 → Phase 4에서 소비 확인
- [ ] OOM 2회 연속 → repair 루프 대신 적응 제안 표시
- [ ] Phase 4 완료 후 최대 3개 후속 실험 제안, 선택 시 최소 재시작 지점부터 실행

---

## 5. Phase 1/4 CrewAI 제거 (별도 작업)

Sprint 1~4와 병행하되, Sprint 2 완료 후 진행 권장.

### Phase 1 전환: forced JSON output 패턴

```python
# 기존: CrewAI Designer 에이전트
# 변경: tool_choice 강제로 schema 준수 JSON 반환

def run_designer_direct(planner_result: PlannerResult, llm_client) -> DesignerResultV4:
    tools = [{
        "name": "submit_design",
        "description": "Submit the final file structure design",
        "input_schema": DesignerResultV4.model_json_schema(),
    }]
    
    response = llm_client.messages.create(
        model=DESIGNER_MODEL,
        tools=tools,
        tool_choice={"type": "tool", "name": "submit_design"},
        messages=[{"role": "user", "content": build_designer_prompt(planner_result)}],
    )
    
    return DesignerResultV4.model_validate(response.content[0].input)
```

**효과**: JSON 파싱 실패 원천 제거, `json_extractor.py` fallback 불필요

### Phase 4 전환: 섹션별 forced JSON + grounding 검증

```python
def write_section_direct(
    section_spec: SectionSpec,
    writer_context: WriterContext,
    llm_client,
) -> SectionResult:
    
    tools = [{
        "name": "submit_section",
        "input_schema": SectionContent.model_json_schema(),
    }]
    
    response = llm_client.messages.create(
        model=WRITER_MODEL,
        tools=tools,
        tool_choice={"type": "tool", "name": "submit_section"},
        messages=[{"role": "user", "content": build_section_prompt(
            section_spec, writer_context
        )}],
    )
    
    content = SectionContent.model_validate(response.content[0].input)
    
    # Grounding 검증: 언급된 수치가 실제 metrics에 있는가
    validate_metric_grounding(content.text, writer_context.metrics)
    
    return SectionResult(section=section_spec.name, content=content.text)
```

---

## 6. 신규 파일 목록

```
crewai_prototype/
├── core/
│   ├── checkpoint.py          # Sprint 1
│   ├── error_classifier.py    # Sprint 1
│   ├── repair_strategy.py     # Sprint 2
│   ├── pipeline_memory.py     # Sprint 2
│   ├── dep_graph.py           # Sprint 2
│   ├── context_compressor.py  # Sprint 2
│   ├── token_budget.py        # Sprint 3
│   ├── run_summary.py         # Sprint 3
│   ├── semantic_validator.py  # Sprint 3
│   ├── failure_patterns.py    # Sprint 4
│   ├── extension_proposer.py  # Sprint 4
│   └── preflight.py           # Sprint 4
├── orchestration/
│   ├── injection_queue.py     # Sprint 4
│   ├── extension_router.py    # Sprint 4
│   └── adaptation_gate.py     # Sprint 4
```

### 기존 파일 수정 범위

| 파일 | Sprint | 변경 내용 |
|------|--------|---------|
| `phases/phase2_coding.py` | 1, 2, 3 | CheckpointManager 통합, PipelineMemory 교체, RepairStrategyRotator 통합 |
| `phases/phase3_execution.py` | 1, 3, 4 | CheckpointManager, 스트리밍 실행, FailurePatternDetector |
| `phases/phase1_planning.py` | Phase 1/4 전환 | CrewAI 제거, forced JSON 호출 |
| `phases/phase4_writing.py` | Phase 1/4 전환, 2 | CrewAI 제거, ContextCompressor 입력 |
| `main.py` | 1, 4 | 체크포인트 복원 로직, PreflightClarifier, ExtensionProposer |
| `orchestration/approval_registry.py` | 4 | InjectionQueue 소비 통합 |

---

## 7. 안정화 검증 시나리오

Sprint 4 완료 후 아래 5개 시나리오를 순서대로 통과해야 최종 안정화로 간주.

### 시나리오 1: 정상 실행 (기준선)
- 입력: "CIFAR-100 ResNet-18 vs ViT-Tiny 분류 비교"
- pre-flight 3개 질문 답변
- 개입 없이 Phase 4까지 완주
- 기대: 논문 초안 완성, run_summary.json 생성

### 시나리오 2: Phase 2 중간 재시작
- Phase 2 15번째 파일 생성 직후 프로세스 강제 종료
- 재시작
- 기대: 16번째 파일부터 시작, 15번째까지 재생성 없음

### 시나리오 3: 환경 오류 즉시 감지
- torch 없는 환경에서 실행
- 기대: repair 루프 없이 즉시 "FATAL_ENV: torch not installed" 메시지

### 시나리오 4: Phase 3 반복 OOM
- GPU 메모리 부족 환경 시뮬레이션 (batch_size를 극대화)
- 기대: 2회 OOM 후 repair 루프 대신 적응 제안 표시

### 시나리오 5: 후속 실험 연장
- 시나리오 1 완료 후 Extension 제안 중 "Data Augmentation" 선택
- 기대: Phase 1 재실행 없이 Phase 2 일부 파일만 재생성 후 Phase 3 시작

---

## 8. 리스크와 완화

| 리스크 | 가능성 | 영향 | 완화 방법 |
|--------|------|------|---------|
| Sprint 2 PipelineMemory 리팩터링 중 기존 repair 로직 회귀 | 중 | 높음 | 각 Sprint는 기존 테스트 시나리오 1 통과 후 머지 |
| Phase 1/4 CrewAI 제거 후 JSON 포맷 오류 | 중 | 중간 | forced JSON tool_choice + Pydantic validation으로 원천 방지 |
| 체크포인트 파일과 workspace 불일치 | 낮음 | 높음 | 체크포인트 로드 시 workspace 파일 존재 여부 재검증 |
| Extension Router가 잘못된 재시작 지점 선택 | 중 | 중간 | 기본값: Phase 2 전체 재실행 (안전 방향) |
| Pre-flight 질문이 실험 방향을 과도하게 제한 | 낮음 | 중간 | MAX_QUESTIONS=4 하드캡, default는 항상 현재 동작과 동일하게 |

---

*작성: Alex (Codex) × Jordan (Claude Code) — `expert_review_crewai_stability_ko.md` 전체 검토 세션 종합*  
*다음 단계: Sprint 1부터 순서대로 구현, 각 Sprint 완료 기준 통과 확인 후 다음 진행*
