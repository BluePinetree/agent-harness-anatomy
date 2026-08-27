# 구현 계획 구조 검토 — 복잡성 감사

> **목적**: `implementation_plan_crewai_stability_ko.md`에 제안된 설계가 실제 코드 흐름과 맞는지,  
> 새 복잡성이 기존 문제보다 더 큰 문제를 만들지 않는지 검토  
> **방법**: 실제 crewai_prototype 코드 구조 감사 결과와 계획서를 대조  
> **작성일**: 2026-05-28

---

## 먼저 확인: 현재 코드 구조의 사실

코드 감사에서 확인한 현재 실행 흐름:

```
main()
  └─ PipelineOrchestrator._execute()          [배경 스레드]
      ├─ Phase 0: setup_workspace()
      ├─ Phase 1: run_planning_phase()
      │   └─ for round in range(MAX_REPLAN_ROUNDS):
      │       └─ ApprovalGate.wait(timeout)
      ├─ Phase 2: run_coding_phase()
      │   └─ for stage in (1, 2, 3):          ← 루프 1
      │       └─ for file_spec in files:      ← 루프 2
      │           └─ _repair_loop():
      │               └─ while True:          ← 루프 3
      │                   └─ GuidanceGate.wait(timeout)
      ├─ Phase 3: run_execution_phase()
      │   └─ while True:                      ← 루프 1
      │       └─ AnalyzerAgent + RepairAgent (CrewAI)
      └─ Phase 4: run_writing_phase()
```

**현재 최대 루프 중첩: 3단계 (Phase 2)**  
**현재 함수 호출 깊이: 약 10~12 레벨**  
**데이터 흐름: 단방향 (역참조 없음)** ✓  
**캔슬 토큰: 모든 루프에서 체크됨** ✓

---

## 문제 1: `DependencyInvalidationGraph`가 새로운 무한 루프를 만든다

### 계획서 내용

```python
# 파일 A repair 후:
stale_files = dep_graph.invalidated_by(file_result.path)
for stale_path in stale_files:
    recheck_queue.add(stale_path)
```

### 구조적 문제

Phase 2의 현재 루프가 `for stage → for file → while repair`인데,  
`recheck_queue`를 도입하면 루프 구조가 다음으로 바뀐다:

```
for stage:
    for file:                         ← 루프 2
        while repair:                 ← 루프 3
            # file A 수정됨
            → recheck_queue에 B, C 추가
    while recheck_queue not empty:    ← 루프 4 (신규)
        for stale_file in queue:      ← 루프 5 (신규)
            while repair:             ← 루프 6 (신규)
                # B 수정됨
                → recheck_queue에 A 재추가???
```

**`invalidated_by()`의 visited set은 그래프 순환을 막지만, repair 자체가 파일을 바꾸기 때문에 "B를 고쳤더니 A가 다시 틀렸다"는 상황이 발생할 수 있다.** 특히 circular import가 있는 파일 그룹에서 이 패턴이 계속 반복된다.

현재 3단계 중첩이 6단계로 늘어나고, 탈출 조건이 불명확하다.

### 대안 (훨씬 단순)

"파일 생성 완료 후 전체 import 검증 1회 패스"로 충분하다:

```python
def _verify_all_imports_once(files_written: list[str], workspace_root: str) -> list[FileResult]:
    """Phase 2 완료 후 단 1회만 실행. 추가 repair loop 없음."""
    issues = []
    for path in files_written:
        check = check_import(path, workspace_root)
        if not check.passed:
            # repair 하지 않음 — 문제 목록만 기록
            issues.append(FileIssue(path=path, error=check.error))
    return issues
    # 이슈가 있으면 smoke test repair loop에서 처리됨 (이미 존재)
```

무한 재검사 루프 없이, 완료 후 1회 스캔만.  
**DependencyInvalidationGraph: 제거 권장**

---

## 문제 2: `PipelineMemory` 4계층이 2줄짜리 수정을 100줄 리팩터링으로 만든다

### 실제 문제

코드 감사에서 확인한 `_repair_content()` (phase2_coding.py):

```python
parts = [
    f"Fix the following Python file: {file_spec.path}",
    f"Error: {check.error}",
    f"Previous content:\n{previous_content}",  # ← 항상 포함
    ...
]
# 3번째 repair 시: 에러 1 + 수정 시도 1 + 에러 2 + 수정 시도 2 + 에러 3 이 누적됨
```

### 계획서 해법

`PipelineMemory` (4개 dataclass: `SemanticContext`, `WorkingContext`, `EpisodicSummary`, 메인 클래스) + `build_repair_prompt()` 메서드 신규 작성.

### 실제로 필요한 수정

```python
# _repair_content() 내부 — 변경 2줄
def _repair_content(file_spec, check, previous_content, attempt, ...):
    parts = [
        f"Fix the following Python file: {file_spec.path}",
        f"Responsibility: {file_spec.responsibility}",
        f"Error: {check.error}",                          # 최신 에러만
        # ↓ 이것만 바꾸면 됨
        f"Previous code:\n{previous_content}" if attempt == 1 else
        "Start fresh from the spec above. Do not reference any previous attempt.",
        ...
    ]
```

그리고 repair 호출 시 `error` 파라미터를 항상 최신 것만 전달 (호출부 1줄 수정).

**실제 문제는 함수 2개에서 파라미터 전달 방식의 2줄 수정이다.**  
4개 클래스 + 새 파일 + 리팩터링은 오버엔지니어링.

**PipelineMemory: 제거. 대신 `_repair_content()` 시그니처에 `attempt` 파라미터 추가.**

---

## 문제 3: 검증 단계가 4단계로 쌓여 Phase 2가 O(n³) 구조가 된다

### 계획서가 추가하는 검증 단계

```
기존: syntax check → import check → smoke test
Sprint 3 추가: semantic validator (dry-run)
```

즉 파일 1개당:

```
while repair_attempt:
    생성
    syntax check     ← 실패 시 repair loop 진입
    import check     ← 실패 시 repair loop 진입
                       (smoke test는 전체 Phase 2 완료 후)
Phase 2 완료 후:
    smoke test       ← 실패 시 repair loop 재진입
    semantic dry-run ← 실패 시 repair loop 재진입
```

smoke test와 semantic validator 둘 다 실패하면 Phase 2 repair loop가 두 번 더 돌아간다.  
최악의 경우:
- 파일 수: N (20개)
- repair 시도: M (5회)
- 검증 단계: 4개

각 검증 단계 실패가 M번 repair를 유발하면: **O(N × M × 4)** 회 LLM 호출.

### 대안

**smoke test와 semantic validator를 하나의 "exit gate"로 통합**하되, 실패 시 무조건 repair loop로 재진입하지 말고 **GuidanceGate로 즉시 에스컬레이션**:

```python
def _run_exit_gate(workspace_root, plan) -> ExitGateResult:
    """Phase 2 완료 직후 1회만 실행. 실패 시 repair 안 하고 사용자에게 바로 알림."""
    smoke = check_syntax_and_import(plan.entry_point, workspace_root)
    if not smoke.passed:
        return ExitGateResult(passed=False, error=smoke.error, source="smoke")
    
    semantic = run_dry_run(workspace_root, plan, max_steps=2, timeout=60)
    if not semantic.passed:
        return ExitGateResult(passed=False, error=semantic.error, source="semantic")
    
    return ExitGateResult(passed=True)

# 실패 시:
# → GuidanceGate ("entry point 실행 오류. 수동 확인이 필요합니다: {error}")
# → 사용자가 hint 제공 시 Phase 2 repair loop 재개
# → 사용자가 skip 시 Phase 3 강제 진입 (현재 동작 유지)
```

파일별 repair loop와 exit gate는 완전히 분리됨. O(N × M)로 유지.

---

## 문제 4: `TokenBudgetTracker`의 전역 사이드이펙트

### 계획서 내용

```python
if pct >= PRESSURE_PCT:
    return BudgetStatus.PRESSURE  # 이후 dep_context 크기 반감
```

호출자가 `_MAX_DEP_CHARS`를 절반으로 줄임.

### 구조적 문제

`_MAX_DEP_CHARS`는 현재 모듈 수준 상수다. 이를 런타임에 변경하면:

- 파일 1~15: 6,000자 dep context로 생성
- 파일 16~20: 3,000자로 생성 (같은 run 내에서 불일치)
- 재시작 후: `_MAX_DEP_CHARS`가 원래 값으로 복원 → 체크포인트와 불일치

디버깅 시 "왜 파일 16부터 import가 안 맞지?"를 추적하기 어렵다.

### 대안

TokenBudgetTracker는 **상태를 emit만 하고 실제 동작 변경은 하지 않는다**:

```python
# TokenBudgetTracker: 관찰만
def record(self, usage: TokenUsage) -> None:
    self.total_input += usage.input_tokens
    pct = self.total_input / self.budget_input
    if pct >= 0.85:
        emit("TOKEN_PRESSURE", {"pct": round(pct * 100)})  # UI 경고만
    elif pct >= 0.70:
        emit("TOKEN_WARNING", {"pct": round(pct * 100)})

# dep_context 크기는 항상 고정 (_MAX_DEP_CHARS 불변)
# 토큰이 부족하면 사용자가 인지하고 중단/계속 결정
```

TokenBudgetTracker는 **모니터링 도구**이지 **행동 변경 트리거**가 아니어야 한다.

---

## 문제 5: `ContextInjectionQueue`가 체크포인트 되지 않는다

### 계획서 내용

Phase 3 실행 중 사용자가 "dropout 추가해봐"를 주입 → 큐에 저장 → Phase 4에서 소비.

### 구조적 문제

Phase 3 실행 도중 프로세스가 종료되면:
- 체크포인트: `phase3_exec_result.json` 없음 → Phase 3 재실행
- InjectionQueue: 메모리 소실 → "dropout 추가해봐"가 사라짐
- Phase 4에서 injection 없이 진행

더 심각한 경우: Phase 3이 **성공** 체크포인트를 저장한 직후 큐 소비 전에 종료되면, 재시작 시 Phase 3을 스킵하고 Phase 4로 가는데 큐가 비어 있어 injection 없이 논문 작성.

### 대안

InjectionQueue를 체크포인트와 같은 경로에 직렬화:

```python
# ContextInjectionQueue.persist() — Phase 경계마다 호출
def persist(self, checkpoint_dir: Path) -> None:
    path = checkpoint_dir / "injection_queue.json"
    path.write_text(json.dumps([e.__dict__ for e in self._queue], ensure_ascii=False))

# 시작 시 복원
def restore(self, checkpoint_dir: Path) -> None:
    path = checkpoint_dir / "injection_queue.json"
    if path.exists():
        entries = json.loads(path.read_text())
        self._queue = [InjectionEntry(**e) for e in entries]
```

---

## 문제 6: `ExtensionRouter`의 "최소 재시작" 로직이 파이프라인 불일치를 만든다

### 계획서 내용

```python
if proposal.modifies_files <= 3:
    return ExtensionRoute(
        restart_from="phase3",
        reuse_workspace=True,
        files_to_regenerate=proposal.affected_files,
    )
```

Phase 2 workspace의 일부 파일만 재생성 후 Phase 3 실행.

### 구조적 문제

Phase 2는 **파일들이 서로의 존재와 API를 알고** 생성된다. `trainer.py`는 `model.py`의 시그니처를 알고, `dataset.py`는 `config.py`의 상수를 알고 쓰여진다.

"Data Augmentation 추가"가 `trainer.py`와 `transforms.py`만 변경한다고 LLM이 판단해도, 실제로는 `dataset.py`의 `__getitem__`도 바꿔야 할 수 있다.

"affects 3 files" 판단 자체가 LLM이 하는데, 이 판단이 틀리면 workspace 불일치로 Phase 3 실패.

가장 나쁜 경우: Phase 3이 새 코드로 실패 → repair loop → "왜 이 함수 시그니처가 다르지?" → 원인 추적 불가.

### 대안 (단순하고 안전)

ExtensionRouter의 판단을 **두 가지만**으로 단순화:

```
if 아키텍처 변경 (모델, 데이터셋, 실험 구조):
    → Phase 1부터 재시작 (완전한 재설계)
else (하이퍼파라미터, augmentation 방법 등):
    → Phase 2 전체 재시작 (workspace 완전 재생성)
```

"Phase 3부터 재시작 + 파일 일부만 교체" 옵션은 제거.  
Phase 2 재시작의 비용: CIFAR-100 기준 15~20분. Extension이 +15분이라면 수용 가능.  
파이프라인 불일치로 Phase 3 실패 후 디버깅하는 시간보다 짧다.

---

## 기존 버그: smoke test 루프의 무한 폴링 (Sprint 1에서 수정 필요)

코드 감사에서 발견한 **계획서에 없는 기존 문제**:

```python
# phases/phase2_coding.py (smoke test 부분)
while True:
    attempt += 1
    if attempt > MAX_AUTO_REPAIR_ATTEMPTS:
        # escalate
    if check.passed:
        break
    time.sleep(2)  # ← 탈출 조건이 check.passed뿐, 이게 안 되면 무한 대기
```

`check.passed`가 외부 이벤트(사용자 수동 수정)에 의해서만 참이 될 수 있는 상황이라면 무한 루프.  
`attempt > MAX_AUTO_REPAIR_ATTEMPTS` 조건이 escalation 후 재진입 시 초기화된다면 탈출 불가.

**Sprint 1에 추가해야 할 수정**:

```python
# 명시적 최대 대기 시간 + 탈출 조건 강화
start = time.time()
while True:
    if time.time() - start > SMOKE_TEST_TIMEOUT_SECS:  # 예: 300초
        emit("SMOKE_TEST_TIMEOUT", {})
        break
    attempt += 1
    ...
```

---

## 종합 판정표

| 계획서 항목 | 판정 | 이유 | 대안 |
|-----------|------|------|------|
| `CheckpointManager` | ✅ 유지 | 명확한 입/출력, 독립적 | — |
| `ErrorClassifier` | ✅ 유지 | 패턴 매칭만, 부작용 없음 | — |
| 컨텍스트 잘림 명시 | ✅ 유지 | 3줄 수정 | — |
| `RepairStrategyRotator` | ✅ 유지 (단순화) | attempt 기반 전략 선택은 올바름 | 단독 클래스 불필요, `_repair_content()` 파라미터 추가로 통합 |
| `ContextCompressor` (Phase 경계) | ✅ 유지 | Phase 4 Writer 입력 압축 필수 | — |
| `FailurePatternDetector` | ✅ 유지 | 패턴 매칭만, 단순 | — |
| `PreflightClarifier` | ✅ 유지 | 배치와 완전 분리, 독립적 | — |
| `TokenBudgetTracker` | ⚠️ 수정 | 모니터링만, 동작 변경 제거 | emit만 하고 `_MAX_DEP_CHARS` 변경하지 않음 |
| `ContextInjectionQueue` | ⚠️ 수정 | 체크포인트와 동기화 필요 | `persist()`/`restore()` 추가 |
| `PipelineMemory` (4계층) | ❌ 제거 | `_repair_content()` 2줄 수정으로 충분 | `attempt` 파라미터 추가, 에러 누적 제거 |
| `DependencyInvalidationGraph` | ❌ 제거 | 새 중첩 루프 → 복잡성 폭발 위험 | Phase 2 완료 후 import 1회 검증 패스 |
| `SemanticValidator` (별도 repair loop) | ⚠️ 수정 | smoke test와 통합, repair loop 재진입 금지 | `ExitGate` 단일 함수, 실패 시 GuidanceGate 직행 |
| `ExtensionRouter` (파일 일부 교체) | ❌ 제거 | workspace 불일치 위험 | Phase 1 또는 Phase 2 전체 재시작만 허용 |
| `PlanAdaptationGate` | ✅ 유지 | FailurePatternDetector의 출력 소비 | — |

---

## 수정된 컴포넌트 목록

제거하거나 수정하면 실제로 필요한 **신규 파일은 7개**다 (계획서의 14개 → 절반):

```
crewai_prototype/
├── core/
│   ├── checkpoint.py          # 유지 (Sprint 1)
│   ├── error_classifier.py    # 유지 (Sprint 1)
│   ├── context_compressor.py  # 유지 (Sprint 2)
│   ├── token_budget.py        # 수정 (모니터링만, Sprint 3)
│   ├── run_summary.py         # 유지 (Sprint 3)
│   ├── failure_patterns.py    # 유지 (Sprint 4)
│   └── preflight.py           # 유지 (Sprint 4)
├── orchestration/
│   ├── injection_queue.py     # 수정 (persist 추가, Sprint 4)
│   └── adaptation_gate.py     # 유지 (Sprint 4)
```

**기존 파일 수정만으로 해결되는 항목:**

| 문제 | 수정 위치 | 변경량 |
|------|---------|------|
| repair 전략 rotation | `phase2_coding.py`: `_repair_content()` 파라미터 추가 | ~10줄 |
| 이전 에러 누적 | `phase2_coding.py`: 호출부에서 최신 에러만 전달 | ~3줄 |
| 잘림 명시 | `phase2_coding.py`: `_build_dep_context()` 내부 | ~5줄 |
| smoke test 무한 루프 | `phase2_coding.py`: 타임아웃 추가 | ~5줄 |
| smoke+semantic 통합 | `phase2_coding.py`: `_run_exit_gate()` 함수 추가 | ~30줄 |
| Extension 단순화 | `main.py`: extension 분기 로직 | ~20줄 |

---

## 최종 의견

**잘 설계된 부분 (건드리지 않아도 됨)**:
- Phase 간 단방향 데이터 흐름
- `CancellationToken`의 모든 루프 내 체크
- `threading.Event` 기반 Gate 메커니즘
- Phase별 독립성

**계획서에서 교정이 필요한 핵심 2가지**:

1. `DependencyInvalidationGraph` → **제거**. 새로운 중첩 루프가 아니라 Phase 2 완료 후 1회 검증.

2. `PipelineMemory` (4계층 구조) → **제거**. `_repair_content()`의 파라미터 수정으로 동일한 효과. 파일 1개, 변경 10줄.

이 두 가지를 제거하면 Sprint 2의 복잡도가 절반 이하로 줄어들고, 기존 3단계 루프 중첩이 유지된다.

---

*기준 코드: `phases/phase2_coding.py` (repair loop 라인 175-337), `orchestration/approval_registry.py` (Gate 메커니즘)*
