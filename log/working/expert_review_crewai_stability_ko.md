# 장기 연구 과제 안정화 — 전문가 설계 검토

> **형식**: Codex 설계 엔지니어(Alex)와 Claude Code 설계 엔지니어(Jordan)의 가상 대화  
> **대상**: `crewai_prototype`의 현재 구현 코드를 기반으로, 스크립트가 길거나 많고 수십 분~수 시간이 걸리는 장기 연구 과제를 **중단 없이 안정적으로 수행**하기 위해 필요한 개선점 도출  
> **작성일**: 2026-05-26

---

## 배경

현재 `crewai_prototype`은 아래 파이프라인을 구현하고 있다.

```
Phase 0 (Workspace)
  → Phase 1 (Planner + Designer, CrewAI)
  → Phase 2 (Coder, 직접 LLM 호출 + repair loop)
  → Phase 3 (Executor, subprocess + AnalyzerAgent)
  → Phase 4 (Writer, CrewAI + 섹션별 revision loop)
```

Phase 2는 2026-05-21에 CrewAI 에이전트를 제거하고 직접 LLM 호출 방식으로 전환해 파일 미생성 문제를 해결했다. 그러나 **샘플 시나리오(5~10개 파일, 5분)**가 아닌, **20~30개 파일을 생성하고 CIFAR-100 분류 실험을 30~60분 실행하는 장기 과제**에서는 여전히 해결되지 않은 구조적 취약점이 존재한다.

---

## 대화

### 1부: 체크포인트 — "실패는 언제든 일어난다"

---

**Alex (Codex 관점):**  
Codex에서 가장 먼저 배운 교훈은 하나였어. *장기 작업은 반드시 중간에 실패한다.* 네트워크 끊김, API 타임아웃, OS 메모리 부족, 혹은 사용자가 실수로 터미널을 닫는 경우까지. Phase 2에서 25개 파일을 생성하는 데 20분이 걸렸는데, Phase 3 실행 중 subprocess가 OOM으로 죽으면 어떻게 되지?

**Jordan (Claude Code 관점):**  
현재 구현에서는 Phase 0부터 다시 시작해야 해. `CodingResult` 객체가 메모리에만 있고, Phase 간 직렬화된 체크포인트가 없거든. `handoff/planner_result.json`과 `handoff/designer_result.json`은 파일로 저장되는데, **`coding_result.json`은 없어.** Phase 3 입력으로 `CodingResult`를 넘기지만 파일로 영속화하지 않는다.

**Alex:**  
Codex에서는 각 태스크 완료 시 즉시 파일로 커밋했어.

```python
# Codex 패턴: 각 단계 결과를 즉시 디스크에 기록
checkpoint_path = workspace / "checkpoints" / f"phase{n}_result.json"
checkpoint_path.write_text(result.model_dump_json(indent=2))
```

그리고 재시작 시 체크포인트가 존재하면 해당 Phase부터 재개:

```python
if (checkpoint := try_load_checkpoint("phase2")):
    coding_result = CodingResult.model_validate_json(checkpoint)
    log.info("Phase 2 결과 캐시에서 복원, Phase 3로 건너뜀")
    return run_phase3(coding_result, ...)
```

**Jordan:**  
Claude Code의 `queryLoop()`도 동일한 철학이야. `State` 객체의 `messages` 배열 전체가 불변 스냅샷으로 각 iteration에 전달돼. 복구가 필요하면 이전 State로 롤백할 수 있어. *상태를 명시적으로 외부화(externalize)하지 않으면 복구는 불가능해.*

**개선 필요: `CheckpointManager`**

```
outputs/<run_id>/checkpoints/
  phase0_workspace.json       # 항상 생성
  phase1_plan_bundle.json     # Phase 1 완료 시
  phase2_coding_result.json   # Phase 2 완료 시 (파일별 FileResult 포함)
  phase3_exec_result.json     # Phase 3 완료 시
  phase2_file_<hash>.done     # 파일 단위 세분화 체크포인트
```

특히 `phase2_file_<hash>.done`은 파일 단위 재시작을 가능하게 한다. 20개 파일 중 17개를 만든 뒤 실패했다면 나머지 3개만 생성하면 된다.

---

### 2부: Repair 전략 — "같은 프롬프트로 같은 실패를 반복한다"

---

**Jordan:**  
현재 `_repair_content()` 로직을 보면, 모든 재시도가 본질적으로 동일한 프롬프트 구조를 사용해:

```python
parts = [
    f"Fix the following Python file: {file_spec.path}",
    f"Error: {check.error}",
    f"Previous content:\n{previous_content}",
    ...
]
```

Claude Code에서 배운 것 중 하나는 **동일한 접근으로 3번 이상 실패하면 접근 자체를 바꿔야 한다**는 거야. `withRetry.ts`에서도 연속 `529` 오류가 3번 발생하면 모델 자체를 폴백해버려.

**Alex:**  
Codex에서는 이걸 "repair strategy rotation"이라고 불렀어. 각 재시도마다 다른 전략을 적용하는 거야:

| 시도 | 전략 | 설명 |
|------|------|------|
| 1 | `targeted_fix` | 에러 라인만 수정, 나머지 유지 |
| 2 | `rewrite_from_spec` | 이전 코드 무시, spec에서 새로 생성 |
| 3 | `minimal_stub` | 최소 구현 + TODO 주석 |
| 4+ | `user_escalation` | GuidanceGate |

특히 `rewrite_from_spec` 전략이 중요해. 기존 코드가 이미 잘못된 방향으로 작성됐을 때, 그 코드를 repair 프롬프트에 포함시키면 LLM이 같은 실수를 반복하거든.

**Jordan:**  
현재 코드에서 `_repair_content()`는 항상 `previous_content`를 포함시켜. 이게 "오염된 컨텍스트로 수정 시도"의 전형적인 패턴이야. 의존성 컨텍스트(`_build_dep_context`)도 재시도마다 재계산되는데, 만약 의존 파일 자체에 에러가 있다면 같은 잘못된 정보를 계속 주입하는 거야.

**개선 필요: `RepairStrategyRotator`**

```python
class RepairStrategy(Enum):
    TARGETED_FIX = "targeted_fix"       # 1번째 시도: 에러 라인만
    FRESH_FROM_SPEC = "fresh_from_spec" # 2번째: 이전 코드 제외
    MINIMAL_STUB = "minimal_stub"       # 3번째: 최소 구현
    USER_ESCALATION = "user_escalation" # 4번째+: GuidanceGate

def select_repair_strategy(attempt: int, error_type: str) -> RepairStrategy:
    if error_type == "syntax" and attempt == 1:
        return RepairStrategy.TARGETED_FIX
    if attempt == 2:
        return RepairStrategy.FRESH_FROM_SPEC
    if attempt >= 3:
        return RepairStrategy.MINIMAL_STUB
    return RepairStrategy.TARGETED_FIX
```

---

### 3부: 오류 분류 — "모든 오류를 동일하게 취급하면 안 된다"

---

**Alex:**  
Codex 인프라에서 오류를 초기에 단일 `try/except`로 묶어서 처리했다가 큰 문제를 경험했어. `SyntaxError`는 즉시 고칠 수 있지만, `ImportError: No module named 'torch'`는 환경 설정 문제라서 아무리 파일을 수정해도 해결 안 돼.

현재 `crewai_prototype`의 `CheckResult`는 다음 세 가지를 구별하긴 해:
```python
error_type: str = ""    # "syntax" | "import" | "runtime" | ""
```

그런데 repair loop에서 이를 실제로 다르게 처리하는 로직이 없어. `error_type`에 따라 repair 전략이 달라져야 해.

**Jordan:**  
Claude Code 관점에서도 오류를 세 범주로 나눠:

1. **Transient**: API 오버로드, 네트워크 타임아웃 → 재시도 (지수 백오프)  
2. **Recoverable**: 코드 오류, 잘못된 로직 → LLM 재생성  
3. **Fatal**: 환경 설정 오류, 누락된 의존성, 사용자 취소 → 즉시 에스컬레이션

현재 코드에서 `ImportError: No module named 'numpy'`가 나면 repair 루프를 3회 돌다가 GuidanceGate로 가. 그런데 이건 LLM이 절대 고칠 수 없는 종류의 오류야. 즉시 "실험 환경에 `numpy`가 없습니다"라고 사용자에게 알려야 해.

**개선 필요: `ErrorClassifier`**

```python
class ErrorClass(Enum):
    TRANSIENT = "transient"         # 재시도
    RECOVERABLE_CODE = "code"       # LLM repair
    RECOVERABLE_LOGIC = "logic"     # fresh_from_spec
    FATAL_ENV = "fatal_env"         # 즉시 에스컬레이션
    FATAL_SEMANTIC = "fatal_sem"    # 의미 오류, 사용자 판단 필요

def classify_error(check: CheckResult, attempt: int) -> ErrorClass:
    if check.error_type == "syntax":
        return ErrorClass.RECOVERABLE_CODE
    if check.error_type == "import":
        if _is_stdlib_or_installed(check.error):
            return ErrorClass.FATAL_ENV  # pip install 필요
        return ErrorClass.RECOVERABLE_CODE  # 파일 경로 오류
    if check.error_type == "runtime":
        if _is_oom(check.error):
            return ErrorClass.FATAL_ENV
        return ErrorClass.RECOVERABLE_LOGIC
    return ErrorClass.RECOVERABLE_CODE
```

---

### 4부: 컨텍스트 오염 — "고치려는 맥락이 실패의 원인이다"

---

**Jordan:**  
이게 장기 실행에서 가장 교묘한 문제야. Phase 2에서 파일 A를 먼저 생성할 때 올바른 컨텍스트를 주지만, 파일 A가 나중에 수정되면 파일 B는 이미 "구 버전 A"를 참고해서 생성된 상태야.

현재 `_build_dep_context()`는 파일 생성 **당시**의 내용을 스냅샷으로 사용해. 파일 수가 적을 때는 문제없지만, 30개 파일을 생성하고 Stage 2에서 Stage 1 파일이 수정되면, Stage 2 파일들의 `import` 가정이 깨진다.

**Alex:**  
Codex에서는 이를 "dependency invalidation graph"로 해결했어. 파일 A가 수정되면 A를 `import`하는 모든 파일을 검사 큐에 다시 추가하는 거야:

```python
class DependencyGraph:
    def __init__(self, file_specs: list[FileNodeSpec]):
        # imports_from 관계에서 역방향 그래프 생성
        self.dependents: dict[str, set[str]] = {}
        for spec in file_specs:
            for dep in spec.imports_from:
                self.dependents.setdefault(dep, set()).add(spec.path)
    
    def invalidated_by(self, modified_path: str) -> set[str]:
        """수정된 파일에 의존하는 모든 파일 반환 (전이적)"""
        result = set()
        queue = [modified_path]
        while queue:
            current = queue.pop()
            for dep in self.dependents.get(current, []):
                if dep not in result:
                    result.add(dep)
                    queue.append(dep)
        return result
```

파일 A가 repair로 변경되면 `invalidated_by(A)`를 계산하고, 그 파일들의 `check` 상태를 `needs_recheck`로 표시한다.

**Jordan:**  
그리고 의존성 컨텍스트 전달 한도(`_MAX_DEP_CHARS = 6,000`)가 있는데, 이게 조용히 잘릴 때가 문제야. 현재는 잘려도 LLM이 알 수 없어. 잘렸다는 신호를 명시적으로 줘야 해:

```python
if len(content) > _MAX_DEP_CHARS:
    content = content[:_MAX_DEP_CHARS]
    content += f"\n\n# [TRUNCATED at {_MAX_DEP_CHARS} chars — only public API guaranteed]"
```

---

### 5부: 토큰 예산 — "모르고 있다가 갑자기 중단된다"

---

**Jordan:**  
Claude Code의 `tokenBudget.ts`에서 배운 핵심 설계 원칙은 **토큰 소비를 각 API 호출 후 즉시 추적하고, 임계값 도달 전에 행동을 바꾼다**는 거야. 현재 `crewai_prototype`에는 Phase 2에서의 토큰 소비 추적이 없어.

```typescript
// Claude Code 패턴
const pct = Math.round((turnTokens / budget) * 100)
if (pct > 70) {
    // 프롬프트에 "be concise" 힌트 추가
    addBudgetPressureHint(messagesForQuery)
}
if (pct > 90 || isDiminishingReturns()) {
    // 중단 결정
    return { action: 'stop', ... }
}
```

**Alex:**  
Codex에서도 코드 생성 시 토큰 소비가 예상보다 2~3배 커지는 케이스가 있었어. 특히 의존성 컨텍스트를 많이 넣을수록 생성되는 코드도 길어지는 경향이 있어. 파일 20개를 만들면서 각 파일마다 6,000자씩 의존성 컨텍스트를 넣으면, 총 입력이 엄청나게 커지거든.

현재 `config.yaml`에 `context_token_budget: 6000`이 있는데, 이게 실제 API 호출당 사용 토큰을 제한하는 게 아니라 CrewAI context 크기 설정이야. Phase 2는 CrewAI를 쓰지 않으므로 이 설정의 영향을 받지 않아.

**개선 필요: `TokenBudgetTracker` (Phase 2 전용)**

```python
@dataclass
class PhaseTokenUsage:
    phase: str
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    calls: int = 0
    budget_warnings: list[str] = field(default_factory=list)
    
class TokenBudgetTracker:
    WARN_AT_PCT = 0.70
    STRATEGY_CHANGE_AT_PCT = 0.85
    
    def record_call(self, usage: TokenUsage):
        self.total += usage.input_tokens + usage.output_tokens
        pct = self.total / self.budget
        
        if pct > self.STRATEGY_CHANGE_AT_PCT:
            # 이후 프롬프트를 최소화 모드로 전환
            self.mode = "minimal"  
            emit("TOKEN_PRESSURE", {"pct": pct})
        elif pct > self.WARN_AT_PCT:
            emit("TOKEN_WARNING", {"pct": pct})
```

---

### 6부: 병렬화 — "순차 생성은 선형으로 느리다"

---

**Alex:**  
현재 Phase 2는 모든 파일을 Stage 1 → Stage 2 → Stage 3 순서로 **순차적으로** 생성해. Stage 1 안에서도 파일들이 서로 의존하지 않으면 병렬로 만들 수 있는데 그냥 순차로 가고 있어.

CIFAR-100 실험에서 파일 구조가 전형적으로:
```
Stage 1: config.py, constants.py        (독립)
Stage 2: dataset.py, transforms.py     (Stage 1 의존)
Stage 3: model.py, trainer.py, eval.py (Stage 1,2 의존)
```

Stage 1의 `config.py`와 `constants.py`는 서로 독립이야. 병렬로 만들면 절반 시간에 끝나.

**Jordan:**  
Claude Code의 `partitionToolCalls()`가 이걸 정확히 해. `isConcurrencySafe()`를 체크해서 독립적인 도구 호출은 묶어서 병렬 실행하거든. 그리고 동시 실행 수를 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`로 제한해.

Phase 2에 같은 원칙 적용:

```python
async def run_coding_phase_parallel(plan: PlanBundle, ...) -> CodingResult:
    dep_graph = DependencyGraph(plan.designer.files)
    
    for stage_files in topological_stages(dep_graph):
        # 같은 Stage 안에서 독립 파일들은 병렬 생성
        tasks = [
            asyncio.create_task(_repair_loop_async(f, workspace_root, llm))
            for f in stage_files
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        # 결과 수집 및 실패 처리
```

**주의사항**: 병렬 생성 시 동일 LLM 제공자의 rate limit에 걸릴 수 있어. `asyncio.Semaphore(max_concurrent=3)`로 동시 LLM 호출 수를 제한해야 해.

---

### 7부: Phase 비대칭 — "Phase 1/4는 아직도 CrewAI에 의존한다"

---

**Jordan:**  
Phase 2가 직접 LLM 호출로 전환된 이유를 다시 보면 — "LLM이 도구를 호출하지 않고 텍스트로 반환하는 문제". 그런데 Phase 1 (Planner, Designer)과 Phase 4 (Writer)는 아직 CrewAI 에이전트를 쓰고 있어. 동일한 문제가 언제든 재발할 수 있어.

Phase 1에서 Designer가 `DesignerResultV4` JSON을 반환해야 하는데, CrewAI 에이전트가 포맷 오류 있는 JSON을 텍스트로 반환하면 `json_extractor.py`의 fallback 파싱에 의존하게 돼. `_strip_fences()` 같은 방어 로직이 있긴 하지만 장기 과제에서 설계 JSON이 길수록 잘릴 위험이 높아.

**Alex:**  
Codex에서는 구조화된 출력이 필요한 에이전트는 항상 **`response_format={"type": "json_schema", ...}`** 모드를 사용했어. Anthropic API에서는 `tool_choice="any"`와 단일 schema tool을 사용해서 강제로 JSON을 반환하게 하는 패턴을 썼어:

```python
# Phase 1 Designer를 직접 LLM 호출로 전환하는 패턴
tools = [{
    "name": "submit_design",
    "description": "Submit the final file design",
    "input_schema": DesignerResultV4.model_json_schema()
}]

response = client.messages.create(
    model=model,
    tools=tools,
    tool_choice={"type": "tool", "name": "submit_design"},  # 강제
    messages=[{"role": "user", "content": design_prompt}]
)
design_result = DesignerResultV4.model_validate(
    response.content[0].input
)
```

이렇게 하면 JSON 파싱 실패가 원천적으로 불가능해.

**Jordan:**  
Phase 4 Writer도 마찬가지야. 현재는 섹션별 텍스트를 생성하고 `_score_section()`으로 품질을 평가하는데, 스코어링 로직 자체가 단어 수나 `[N]` 마크 유무 같은 heuristic이야. 진짜 장기 논문에서는 "결과 수치가 실제 실험 메트릭과 일치하는가"를 검증해야 하는데, 지금은 그게 없어.

```python
# 현재 스코어링의 한계
weights = {
    "result_grounding": 0.35,  # 메트릭 '포함'만 체크, 값이 맞는지는 미검증
    "word_count": 0.30,        # 분량 heuristic
    "citations": 0.20,
    "syntax": 0.15
}
```

---

### 8부: 실행 관찰가능성 — "무슨 일이 일어나는지 모른다"

---

**Alex:**  
장기 실험 실행 중에 사용자가 진행 상황을 볼 방법이 없어. Phase 3에서 30분짜리 훈련 실행이 진행 중일 때 현재 코드는:

```python
result = subprocess.run(
    ["python", entry_point],
    capture_output=True,
    timeout=exec_timeout,
    cwd=workspace_root,
)
```

이건 실행이 끝날 때까지 완전히 블로킹이야. 사용자는 "실행 중"이라는 상태 메시지 외에 아무것도 안 보여. 학습 곡선이 발산하고 있어도, epoch가 10/100인지 50/100인지도 몰라.

**Jordan:**  
Claude Code에서 장기 프로세스를 다룰 때는 **AsyncGenerator 스트리밍**을 사용해. subprocess의 stdout을 실시간으로 읽어서 UI로 스트리밍하거든:

```python
# Phase 3 개선: 실시간 stdout 스트리밍
async def _run_script_streaming(entry_point: str, ...) -> AsyncGenerator[str, None]:
    process = await asyncio.create_subprocess_exec(
        "python", entry_point,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        cwd=workspace_root,
    )
    
    async for line in process.stdout:
        decoded = line.decode("utf-8", errors="replace")
        emit("EXEC_STDOUT", {"line": decoded, "timestamp": time.time()})
        yield decoded
    
    await process.wait()
    return process.returncode
```

그리고 `emit("EXEC_STDOUT", ...)`로 WebSocket을 통해 UI에 실시간으로 전달.

**Alex:**  
또한 실험 중간 메트릭을 파싱할 수 있으면 좋아. 예를 들어 PyTorch 출력에서 `Epoch [10/100], Loss: 1.234` 패턴을 실시간으로 파싱해서 UI에 그래프로 보여주면, 학습이 발산하고 있을 때 사용자가 조기에 중단할 수 있어:

```python
EPOCH_PATTERN = re.compile(
    r"Epoch\s*\[?(\d+)/(\d+)\]?,?\s*[Ll]oss[:\s]+([\d.]+)"
)

def parse_training_line(line: str) -> TrainingMetric | None:
    m = EPOCH_PATTERN.search(line)
    if m:
        return TrainingMetric(
            epoch=int(m.group(1)),
            total_epochs=int(m.group(2)),
            loss=float(m.group(3))
        )
    return None
```

---

### 9부: 우아한 저하 — "실패해도 가치 있는 결과를 남겨야 한다"

---

**Jordan:**  
Claude Code의 핵심 원칙 중 하나는 **"partial completion is still valuable"**이야. 도구 루프가 중단됐을 때 완전히 처음부터 재시작하지 않고, 완료된 부분의 결과를 사용자에게 돌려줘.

현재 crewai_prototype에서 Phase 3가 5번 repair 시도 후 실패하면 `ExecutorResult(success=False)`를 반환하고 Phase 4가 스킵되거나 빈 결과로 Writer를 실행해. 이때 "실험 실패 분석 보고서"라도 써줄 수 있어야 해.

**Alex:**  
Codex에서는 이걸 "degraded output contract"라고 불렀어:

| 상황 | 최소 보장 출력 |
|------|--------------|
| Phase 2 부분 실패 | 생성된 파일들 + stub 목록 + 실패 원인 분석 |
| Phase 3 실행 실패 | 에러 로그 분석 보고서 + 코드 리뷰 섹션 |
| Phase 3 부분 성공 | 중간 에포크 메트릭으로 추세 분석 |
| 전체 실패 | 실패 원인 진단 문서 |

Phase 4 Writer가 `ExecutorResult.success == False`를 받았을 때 "실험 실패 분석" 섹션을 쓰도록 설계되어야 해. 현재는 Writer task description에 실패 케이스가 명시되어 있지 않아.

---

### 10부: smoke test의 한계 — "구문이 맞아도 의미가 틀릴 수 있다"

---

**Alex:**  
Phase 2의 smoke test는 `syntax + import` 검사야. 그런데 다음 케이스를 생각해봐:

```python
# 생성된 experiment_impl.py — syntax OK, import OK
def train(config):
    model = ResNet18(num_classes=10)  # CIFAR-10 클래스 수!
    # ... 나머지 로직
```

CIFAR-100은 클래스가 100개인데 10으로 고정됐어. 구문 오류도 없고 import도 문제없어. smoke test는 통과해. 그런데 Phase 3에서 실행하면 정확도가 극도로 낮게 나오거나 shape 오류가 발생해.

**Jordan:**  
이건 "semantic validation"의 영역이야. Claude Code에서는 파일을 편집한 후 테스트를 실행해서 semantic correctness를 확인해. crewai_prototype에서는 이를 위한 "quick sanity test" 단계를 추가할 수 있어:

```python
# smoke test 확장: API contract 검사
def _check_api_contract(workspace_root: str, plan_bundle: PlanBundle) -> CheckResult:
    """
    코드가 PlannerResult에 명시된 계약을 이행하는지 빠르게 확인.
    - 올바른 num_classes (10, 100 등)
    - entry point 함수 시그니처
    - 필수 출력 파일 경로
    """
    # 간단한 dry-run: --dry-run 인수로 1 epoch만 실행
    result = subprocess.run(
        ["python", entry_point, "--dry-run", "--max-steps", "5"],
        capture_output=True, timeout=60, cwd=workspace_root
    )
    # 결과 JSON에서 num_classes 확인
    ...
```

---

### 11부: 진단 vs. 처방 — "왜 실패했는지를 기록해야 다음에 막는다"

---

**Jordan:**  
현재 `RepairRecord`가 있긴 한데:

```python
class RepairRecord(BaseModel):
    attempt: int
    strategy: str
    error_before: str
    error_after: str
    success: bool
```

`repair_records`가 `CodingResult`에 포함되지만, Phase 3/4로 전달되지 않고 Phase 2 내부에서만 쓰이다가 사라져. 이 데이터가 최종 보고서에 포함되면 연구자가 "이 파일이 왜 여러 번 수정됐고 어떤 오류 패턴이 있었는가"를 알 수 있어.

**Alex:**  
Codex에서 운영하면서 가장 가치 있었던 데이터 중 하나가 "repair 이력"이야. 동일 에러가 반복되는 패턴을 분석해서 프롬프트를 개선했어. 이 데이터를 버리는 건 큰 손실이야.

**개선 필요: `run_summary.json`**

```json
{
  "run_id": "...",
  "topic": "CIFAR-100 image classification",
  "phases": {
    "phase2": {
      "files_generated": 18,
      "files_stubbed": 2,
      "total_repairs": 7,
      "repair_breakdown": {
        "syntax": 3,
        "import": 2,
        "runtime": 2
      },
      "token_usage": {
        "total_input": 84000,
        "total_output": 42000
      }
    },
    "phase3": {
      "attempts": 3,
      "final_metrics": {"accuracy": 0.72, "loss": 1.23},
      "repair_breakdown": {"runtime": 2, "logic": 1}
    }
  },
  "total_duration_sec": 2847,
  "success": true
}
```

---

## 종합 개선 우선순위

두 엔지니어가 논의한 개선사항을 구현 우선순위로 정리한다.

### Tier 1 — 즉시 (안정성에 직결)

| 개선 | 현재 문제 | 예상 효과 |
|------|---------|---------|
| **Phase 간 체크포인트** | 실패 시 처음부터 재시작 | Phase 단위 재시작 가능 |
| **오류 분류기** | 환경 오류도 repair 루프로 소진 | Fatal 오류 즉시 에스컬레이션 |
| **의존성 무효화 그래프** | 파일 수정 후 의존 파일 stale | 수정 후 자동 재검사 |
| **컨텍스트 잘림 명시** | 조용히 잘린 의존성 컨텍스트 | LLM이 잘림 인식 |

### Tier 2 — 단기 (성능 및 품질)

| 개선 | 현재 문제 | 예상 효과 |
|------|---------|---------|
| **Repair 전략 rotation** | 동일 프롬프트 반복 | 실패 패턴 탈출 가능 |
| **Phase 1/4를 직접 LLM 호출로 전환** | CrewAI 에이전트 의존성 잔존 | 파이프라인 전체 동일 신뢰성 |
| **토큰 예산 추적** | Phase 2 토큰 소비 무감지 | 예산 압박 시 전략 전환 |
| **실시간 stdout 스트리밍** | 실행 중 진행 상황 불투명 | 조기 발산 감지 가능 |
| **파일 단위 체크포인트** | Stage 단위까지만 재시작 | 파일 단위 정밀 재시작 |

### Tier 3 — 중기 (연구 품질)

| 개선 | 현재 문제 | 예상 효과 |
|------|---------|---------|
| **우아한 저하 계약** | 실패 시 빈 결과 | 부분 결과도 의미 있게 |
| **Semantic API contract 검사** | 구문만 검사, 의미 오류 미감지 | num_classes 등 설정 오류 조기 발견 |
| **병렬 파일 생성** | 순차 생성으로 느림 | 독립 파일 2~3배 빠르게 |
| **run_summary.json** | repair 이력 사라짐 | 반복 실패 패턴 분석 가능 |
| **학습 메트릭 실시간 파싱** | 실행 결과 사후에만 확인 | 발산 조기 감지, 사용자 개입 |

---

## 결론

**Alex:**  
정리하면, 현재 crewai_prototype은 "5분 샘플"에서 검증된 설계야. Phase 2의 직접 LLM 호출 전환은 탁월한 결정이었어. 하지만 "30분짜리 실험"을 안정적으로 수행하려면 **실패가 필연이라는 전제 하에 설계**해야 해. 체크포인트 없이 30분 짜리 파이프라인을 돌리는 건, 자동 저장 없는 문서 편집과 같아.

**Jordan:**  
Claude Code 관점에서 가장 중요한 원칙은 **"루프는 결코 사용자를 놀라게 해서는 안 된다"**는 거야. 예상치 못한 종료, 진행 상황 없는 긴 침묵, 이해할 수 없는 오류 메시지 — 이 세 가지가 장기 작업에서 신뢰를 무너뜨려.

Tier 1 개선만 완료해도 CIFAR-100 3회 연속 성공 확률이 크게 높아질 거야. Tier 2까지 가면 AutoGen/LangGraph와 공정하게 비교할 수 있는 기반이 생겨.

---

*이 문서는 실제 구현 코드(`phases/phase2_coding.py`, `core/handoff_models.py`, `query.ts` (Claude Code) 등)를 기반으로 작성됐습니다. 제안된 코드 패턴은 의사코드(pseudocode)이며 실제 구현 시 현재 코드 구조에 맞게 조정이 필요합니다.*

---

## 부록: GPT 설계 조언에 대한 추가 검토

> **배경**: 다음 7가지 조언은 GPT가 제안한 "유연하고 안정적인 LLM 에이전트 설계" 원칙이다.  
> Alex와 Jordan이 각 원칙이 `crewai_prototype`에 **적용 가능한지, 적용 시 어떤 형태여야 하는지, 혹은 제외해야 하는지**를 검토한다.

---

### 전제 논의: 도메인 불일치 인식

---

**Jordan:**  
GPT 조언을 읽고 첫 번째로 느낀 건 — 이건 **대화형 에이전트(conversational agent)**에 대한 설계 원칙이야. 예시들이 전부 "이 영상 뭐야? → 모델 측정인가? → 유명인 누가 있어?" 같은 실시간 대화 흐름이거든. crewai_prototype은 그게 아니야.

우리 시스템은 **배치 파이프라인(batch research pipeline)**이야:
- 실행 시작 시 연구 주제가 고정된다
- 사용자 개입은 ApprovalGate와 GuidanceGate 두 지점에서만 일어난다  
- 목표는 실행 도중 방향을 바꾸는 게 아니라, **주어진 방향을 30~60분 동안 안정적으로 완주**하는 것이다

**Alex:**  
맞아. Codex도 마찬가지였어. Codex는 "write a web server in Go"라는 작업을 받으면 그 작업을 끝까지 실행하는 배치 에이전트야. 대화 도중 "아 그냥 Python으로 해줘"라고 바꿀 수 없어.

그래서 GPT 조언 7개를 **단순히 적용하면 안 되고**, 각각이 우리 도메인에서 어떤 의미를 갖는지 변환해서 봐야 해. 일부는 직접 적용, 일부는 변형 적용, 일부는 우리 시스템에서 안티패턴이야.

---

### 검토 1: "계획(Plan)이 아니라 상태(State)"

> *"rigid planning이 아니라 evolving conversational state"*  
> *"conversation_state = { entities, inferred intents, uncertainty, emotional tone, ... }"*

---

**Jordan:**  
이 조언의 핵심 주장은 — 사용자가 대화 중간에 목표를 바꾸니까, 처음 세운 Plan에 고착하지 말고 매 턴마다 상태를 재평가하라는 거야. 대화형 에이전트한테는 맞는 말이야.

그런데 우리 시스템에 그대로 적용하면 **위험**해. Phase 1에서 세운 실험 계획을 Phase 2 도중에 "아 이쪽이 더 나을 것 같은데"라며 바꾸면 파이프라인 전체가 불일치 상태가 돼. Phase 2는 Phase 1의 `DesignerResultV4`를 **신뢰하고** 그대로 구현해야 해.

**Alex:**  
동의해. 그런데 조언에서 건져낼 게 하나 있어 — **uncertainty 추적** 개념. 현재 `PlannerResult`는 실험 계획을 내놓지만, "이 계획이 실행 불가능한 상황이 됐을 때 어떻게 대응하는가"에 대한 구조가 없어.

예를 들어, Phase 3에서 CIFAR-100 학습이 OOM으로 4번 연속 실패하면 — 현재는 그냥 GuidanceGate로 사용자에게 던져버려. 그런데 시스템이 "원래 계획(ViT + ResNet 비교)을 이 환경에서 실행하기 어렵다는 걸 인식"하고 **계획을 자동으로 다운스케일**하는 선택지를 제시할 수 있어야 해.

```python
# "Plan Adaptation Gate" 개념 — Phase 3 반복 실패 시
class PlanAdaptationProposal(BaseModel):
    trigger: str            # "repeated_oom_in_phase3"
    original_plan: str      # "ViT-Base + ResNet-50 on CIFAR-100"
    adapted_plan: str       # "ViT-Tiny + ResNet-18 on CIFAR-10"
    rationale: str          # "GPU 메모리 한도에 맞게 모델/데이터셋 축소"

# Phase 3 repair loop에서 적용
if oom_count >= 3:
    proposal = _generate_plan_adaptation(exec_history, plan_bundle)
    emit("PLAN_ADAPTATION_PROPOSAL", proposal)
    # 사용자 승인 → plan_bundle 업데이트 후 Phase 2부터 재시작
```

**Jordan:**  
정리하면 이 조언의 변환:

| GPT 조언 | 대화형 에이전트 | crewai_prototype 변환 |
|---------|-------------|---------------------|
| evolving state | 매 턴 목표 재평가 | **제외** — 파이프라인 불일치 위험 |
| uncertainty tracking | 대화 모호성 처리 | **변환 적용** — Plan Adaptation Gate (Phase 3 반복 실패 시) |
| emotional tone, entity tracking | 사용자 감정 추적 | **제외** — 배치 파이프라인 무관 |

**판정: 부분 변환 적용.** `evolving conversational state` 개념은 제외. 대신 Phase 3의 환경적 실패 패턴에서 계획을 제한적으로 수정하는 `PlanAdaptationGate`로 변환.

---

### 검토 2: "검색 결과를 그대로 쓰지 않는다"

> *"Query Reformulation → Multi-query retrieval → Ranking → Memory-conditioned retrieval → Context compression → Tool-specific grounding"*

---

**Alex:**  
이 조언은 **RAG(Retrieval-Augmented Generation)** 파이프라인 설계야. "검색 결과를 바로 LLM에 넣지 말고 ranking, dedup, compression해서 넣어라"는 거지.

우리 시스템은 현재 외부 검색을 안 해. Phase 1 Planner가 생성하는 실험 설계는 LLM 내부 지식 기반이지, web search나 논문 DB 검색 기반이 아니야.

**Jordan:**  
그런데 현재 코드에서 "검색"에 해당하는 게 없다고 해도, 조언의 핵심 원리 — **입력 데이터를 LLM에 넣기 전에 항상 전처리/압축해야 한다** — 는 우리 시스템에 이미 부분적으로 구현되어 있어. `_build_dep_context()`의 6,000자 제한이 바로 그거야. 다만 불완전하고, 다른 Phase에는 적용이 안 돼.

Phase 3 결과를 Phase 4 Writer에 넘길 때, `stdout_tail`을 raw로 넘기면 안 돼. 훈련 로그 500줄을 그대로 넣는 게 아니라 핵심 메트릭만 추출해서 넘겨야 해.

**Alex:**  
그리고 이 조언에서 가장 가치 있는 아이디어는 **"memory-conditioned retrieval"** 개념이야 — 이전 실험 결과를 기억하고, 그게 새 프롬프트 구성에 영향을 줘야 한다는 거. 우리 시스템은 매 run이 독립적인데, 동일 연구 주제를 여러 번 돌릴 때 이전 run의 실패 패턴을 참고할 수 없어.

이건 Tier 3 수준의 개선이야. 즉각 필요한 건 아니지만 연구 비교 단계에서 가치가 있을 거야.

**판정: 현재 시스템에는 직접 적용 불가.** 검색 인프라가 없음. 단, "입력 데이터를 LLM 전달 전 압축"하는 원칙은 Phase 경계 데이터 처리에 **이미 구현 필요한 항목**으로, 기존 검토 내용과 겹침. 별도 추가 없음.

---

### 검토 3: "Implicit Intent Tracking"

> *"surface text / latent goal / conversational trajectory를 동시에 추적"*  
> *"이게 안 되면 질문마다 새 task로 인식, coherence 붕괴"*

---

**Jordan:**  
솔직히 말하면, 이건 우리 시스템에 완전히 무관한 조언이야. 배치 파이프라인에서 "latent goal shifting"이나 "conversational trajectory"는 존재하지 않아. 연구 주제는 Phase 0 시작 시 명시적으로 주어지고, 그게 전체 실행의 유일한 intent야.

**Alex:**  
오히려 이 조언을 잘못 적용하면 해로울 수도 있어. Phase 2에서 Coder가 "이 파일 spec의 latent intent를 추론해서 spec에 없는 기능을 추가"하면 안 되거든. Coder는 spec을 그대로 구현해야 해 — 이게 현재 Phase 2의 핵심 설계 원칙이야. LLM이 "더 나은 설계를 추론해서" 자의적으로 파일 구조를 바꾸는 게 가장 큰 실패 패턴 중 하나였어.

**Jordan:**  
맞아. 배치 파이프라인에서 implicit intent tracking을 활성화하면 "agent가 spec을 무시하고 자기 판단으로 재설계"하는 문제로 이어질 수 있어. 우리가 원하는 건 정확히 반대야 — **explicit, traceable, deterministic execution**.

**판정: 제외.** 이 조언은 대화형 에이전트 전용. 배치 파이프라인에서는 implicit intent 추론이 spec drift로 이어지므로 오히려 안티패턴.

---

### 검토 4: "Context Entropy — episodic/semantic/working/archival 메모리 분리"

> *"raw history를 계속 넣지 않는다"*  
> *"일정 시점마다 summarize / distill / compress / forget"*

---

**Alex:**  
이게 GPT 조언 7개 중에서 우리 시스템에 **가장 직접적으로 적용 가능한 항목**이야. 아무 변환 없이 그대로 적용해야 해.

현재 Phase 2 repair loop의 LLM 프롬프트 구조를 생각해봐. 첫 번째 repair 시도:

```
[원본 spec] + [생성된 코드] + [에러 메시지] + "Fix this"
```

세 번째 repair 시도:
```
[원본 spec]
+ [생성된 코드]
+ [에러 1] + [1차 수정 시도] + [에러 2] + [2차 수정 시도] + [에러 3]
+ "Fix this"
```

이게 raw history를 계속 넣는 패턴이야. 3번 repair 후 프롬프트가 original spec의 3~4배로 불어나. 그런데 LLM이 실제로 "에러 1과 2차 수정 시도의 상세 내용"을 다 참조하는 게 아니라 오히려 노이즈가 돼.

**Jordan:**  
GPT가 제안한 4계층 메모리 구조를 우리 파이프라인에 직접 매핑해볼게:

| GPT 계층 | 역할 | crewai_prototype 대응 |
|---------|------|---------------------|
| **working** | 현재 active task | 단일 파일 생성/수정 중인 LLM 프롬프트 |
| **episodic** | 최근 interaction | 현재 Phase 내 생성된 파일 목록 + 검사 결과 |
| **semantic** | 사용자 특성/도메인 지식 | stack rule, PlannerResult의 연구 주제/제약 |
| **archival** | 오래된 로그 | 완료된 Phase의 상세 repair 이력, 전체 stdout |

문제는 현재 이 4개가 분리되지 않고 단일 프롬프트에 뒤섞인다는 거야.

**Alex:**  
Codex 인프라에서 이 분리가 가장 큰 안정성 기여를 했어. 핵심은 **프롬프트에 들어가는 것과 파일로 보관되는 것을 명확히 구분**하는 거야:

```python
class PipelineMemory:
    """Phase 2 실행 중 메모리 계층 관리"""
    
    # Working: 현재 파일 생성에만 필요한 것
    working: WorkingContext = field(default_factory=WorkingContext)
    
    # Episodic: 현재 Phase 내 생성 이력 (요약본만 프롬프트에)
    episodic_summary: str = ""          # "17개 파일 생성 완료, 2개 repair 중"
    episodic_full: list[FileResult] = []  # 디스크에만
    
    # Semantic: Plan에서 추출한 변하지 않는 도메인 지식
    semantic: SemanticContext = field(default_factory=SemanticContext)
    # → stack_rule, num_classes, dataset_name, entry_point_signature
    
    # Archival: 로그 파일 경로만 (내용은 디스크)
    archival_paths: list[str] = []

    def build_repair_prompt(self, file_spec, previous_content, check, attempt) -> str:
        """repair 프롬프트에 필요한 계층만 선택적으로 포함"""
        parts = []
        
        # Semantic은 항상 포함 (작고 중요)
        parts.append(self.semantic.to_prompt_block())
        
        # Working: 현재 파일 spec (항상 포함)
        parts.append(f"File: {file_spec.path}\n{file_spec.responsibility}")
        
        # attempt에 따라 이전 코드 포함 여부 결정
        if attempt == 1:
            # 첫 repair: 이전 코드 포함 (targeted fix)
            parts.append(f"Previous code:\n{previous_content}")
        else:
            # 2번째 이상: 이전 코드 제외 (fresh_from_spec)
            parts.append("# Note: start fresh from spec, ignore previous attempts")
        
        # Episodic: 요약본만 (전체 이력 제외)
        if self.episodic_summary:
            parts.append(f"Pipeline context: {self.episodic_summary}")
        
        # 에러는 최신 1개만 (누적 금지)
        parts.append(f"Current error:\n{check.error}")
        
        return "\n\n".join(parts)
```

**Jordan:**  
그리고 "일정 시점마다 summarize/distill/compress"는 Phase 경계에서 구현해야 해. Phase 2가 끝나고 Phase 3으로 넘어갈 때, `CodingResult`의 전체 `repair_records`를 raw로 넘기면 안 되고, 압축된 요약만 넘겨:

```python
def compress_coding_result_for_handoff(result: CodingResult) -> CodingHandoffSummary:
    """Phase 2 → 3 핸드오프용 압축 요약"""
    return CodingHandoffSummary(
        files_written=len([f for f in result.all_files if f.written]),
        files_stubbed=len([f for f in result.all_files if not f.written]),
        # 전체 repair 이력 → 핵심만 추출
        notable_issues=[
            f"{r.path}: {r.check.error_type} (resolved after {len(r.repair_records)} repairs)"
            for r in result.all_files if r.repair_records
        ],
        smoke_test_passed=result.smoke_test_passed,
        # 실제 파일 내용은 workspace 경로만 (디스크에서 직접 읽기)
        workspace_root=result.workspace_root,
    )
```

**판정: 강력히 적용.** GPT 조언 7개 중 crewai_prototype에 가장 직접적으로 적용 가능. 4계층 메모리 분리 구조를 Phase 2 repair loop와 Phase 경계 핸드오프 두 곳에 즉시 구현해야 함.

---

### 검토 5: "Tool Arbitration — policy/routing/safety/uncertainty layer"

> *"LLM decides tool → 이게 아니라 policy + routing + safety + uncertainty estimation + verifier가 tool 호출을 감싸야 한다"*  
> *"검색할까? 기억 기반으로 답할까? clarification 할까? tool 여러 개 병렬 호출?"*

---

**Jordan:**  
이 조언은 이미 우리 시스템에 **부분적으로 구현**되어 있어. 가장 중요한 부분이 2026-05-21의 Phase 2 개선이었거든.

Phase 2 이전 구조:
```
LLM → decides whether to call WorkspaceWriteTool → (sometimes doesn't call it) → file missing
```

Phase 2 이후 구조:
```
Python orchestrator → LLM (text generation only) → Python writes to disk
```

이게 바로 "LLM이 tool을 결정하지 않도록 policy layer가 감싸는 것"이야. LLM이 "파일을 쓸지 말지"를 결정하지 않아 — Python이 항상 파일을 써.

**Alex:**  
그런데 GPT 조언에서 건져낼 게 하나 있어 — **uncertainty estimation**. "이 파일을 내가 정말 생성할 수 있는가"를 LLM이 명시적으로 표현하고, 그걸 orchestrator가 활용하는 구조.

현재 repair loop는 attempt count로만 전략을 결정해. "3번 시도했으니 GuidanceGate"가 아니라, "LLM이 이 파일을 생성하기 충분한 정보를 갖고 있는가"를 판단할 수 있으면 더 좋아.

구현 방법: 파일 생성 전 LLM에게 pre-flight confidence check 질문:

```python
def _check_llm_confidence(file_spec: FileNodeSpec, dep_context: str, llm) -> float:
    """LLM이 이 파일을 생성할 수 있는지 자기 평가 (0~1)"""
    prompt = f"""
    You are about to write: {file_spec.path}
    Responsibility: {file_spec.responsibility}
    Available dependencies: {dep_context[:500]}
    
    Rate your confidence (0.0-1.0) that you can write this correctly.
    Reply with ONLY a number. If < 0.6, also append: NEED_CLARIFICATION: <what's missing>
    """
    response = llm.call([{"role": "user", "content": prompt}])
    # 0.6 미만이면 → spec 보강 요청 또는 stub 즉시 작성
```

**Jordan:**  
"routing / safety" 레이어는 ErrorClassifier(Tier 1)에서 이미 다뤘어. Phase 3의 Analyzer가 에러 분류를 담당하는 것도 tool arbitration의 한 형태야. 추가 구현 필요한 새 요소는 위에서 말한 uncertainty estimation 정도야.

**판정: 부분 이미 구현.** Policy layer(Python이 파일 I/O 담당)와 routing layer(ErrorClassifier)는 기존 설계에서 다룸. **새로운 기여는 uncertainty estimation 개념** — pre-flight confidence check 형태로 적용 가능. 낮은 우선순위(Tier 3).

---

### 검토 6: "생성보다 압축이 더 중요"

> *"50 search docs → rerank → deduplicate → contradiction detect → saliency extraction → compress → inject"*  
> *"안 그러면 hallucination / distraction / instruction dilution"*

---

**Alex:**  
GPT가 말한 배경은 RAG 파이프라인이지만, 이 조언의 근본 원칙 — **LLM에 주입되는 정보를 압축하는 데 generation만큼의 compute를 써야 한다** — 은 우리 시스템의 핵심 미해결 문제야.

현재 `_build_dep_context()`는 파일 단위 context를 6,000자로 자르지만, 아래 처리는 하지 않아:

- **Deduplication**: 여러 파일이 같은 utility function을 import하면 동일 코드가 여러 번 포함됨
- **Saliency extraction**: 의존 파일 전체를 넣는 게 아니라 "현재 파일이 참조할 public API만" 추출
- **Contradiction detection**: Phase 1 Designer 결과와 Phase 2에서 실제 생성된 코드 간 의도 차이 감지

**Jordan:**  
Phase 4 Writer가 가장 취약해. Writer는 Phase 3 실험 결과, Phase 2 코딩 구조, Phase 1 연구 계획을 모두 참조해야 하는데, 이게 다 raw로 들어가면 instruction dilution이 발생해. "논문 Introduction은 연구 배경을 서술해야 한다"는 지시가 "accuracy: 0.723, loss: 1.23, epoch: 50, batch_size: 128..."로 가득 찬 실험 로그에 묻혀버려.

**Alex:**  
Phase별 context 압축 방법을 구체적으로 설계하면:

```python
class ContextCompressor:
    """Phase 경계에서 LLM 입력 전 context 압축"""
    
    @staticmethod
    def compress_for_writer(
        plan: PlannerResult,
        coding: CodingHandoffSummary,
        exec_result: ExecutorResult,
    ) -> WriterContext:
        return WriterContext(
            # Plan → 연구 목적과 가설만 (설계 세부사항 제외)
            research_objective=plan.objective,
            hypotheses=plan.hypotheses,
            
            # Coding → 구조적 사실만 (수정 이력 제외)
            architecture_summary=f"{coding.files_written}개 파일, "
                                  f"주요 모듈: {', '.join(coding.key_modules)}",
            
            # Execution → 핵심 수치만 (전체 로그 제외)
            key_metrics=exec_result.metrics,  # accuracy, loss 등 수치
            training_trajectory=exec_result.epoch_summary,  # 요약본
            # stdout 전체는 제외
        )
```

**Jordan:**  
이 압축 원칙은 사실 1부에서 다룬 "Phase 경계 핸드오프 압축"과 동일한 맥락이야. GPT 조언이 이를 독립적으로 다시 강조하는 건, 얼마나 중요한지 방증이야.

**판정: 강력히 적용.** 검토 4 (메모리 계층)와 쌍을 이루는 핵심 개선. GPT가 제시한 압축 파이프라인(rerank/dedup/saliency/compress)을 RAG가 아닌 **Phase 경계 데이터 처리**로 변환해서 적용.

---

### 검토 7: "Self-reframing — 매 턴 'What is the actual task now?' 재평가"

> *"초기 identify video → 중간 infer context → 후반 measurement discussion → 마지막 AI architecture meta-discussion"*  
> *"이게 없으면 이전 objective에 fixation, conversational brittleness"*

---

**Jordan:**  
이 조언의 원형은 대화 에이전트용이야. 하지만 배치 파이프라인에서 이에 해당하는 현상이 실제로 발생해 — **Phase 3에서 실험 방향이 현실적으로 불가능해진 상황에서의 fixation** 문제야.

현재 Analyzer는 "코드 오류를 고쳐라"만 판단해. "지금 설정(ViT-Base on CIFAR-100)이 이 환경에서 근본적으로 불가능한가"를 판단하지 않아.

**Alex:**  
Codex에서 이와 유사한 상황이 있었어. "PostgreSQL로 X를 구현해달라"는 작업을 받았는데, 5번 실패 후 분석하니 그 환경에 PostgreSQL이 설치가 안 된 거야. 코드를 계속 고치는 게 아니라 "작업 자체의 전제조건이 충족되지 않음"을 인식해야 했어.

우리 시스템에서 이에 해당하는 상황:

| 반복 실패 패턴 | self-reframing 행동 |
|-------------|---------------------|
| GPU OOM × 3회 이상 | 모델/데이터셋 downscale proposal |
| ModuleNotFound (torch/torchvision) × 2회 이상 | 환경 설정 확인 요청 (실험 중단) |
| 수렴하지 않는 loss × 5 epoch 이상 | 학습률/배치 크기 자동 조정 proposal |
| timeout × 2회 이상 | max_epoch 축소 또는 다른 평가 방법 proposal |

**Jordan:**  
이건 검토 1에서 제안한 `PlanAdaptationGate`와 동일한 개념이야. 다만 GPT 조언이 추가로 강조하는 건 — 이걸 Phase 3 repair loop 내부에서 **자동으로 감지**해야 한다는 거야. 사용자가 GuidanceGate에서 "뭘 어떻게 해야 하나요?"를 막막하게 보는 게 아니라, 시스템이 "이 패턴을 보니 X가 문제인 것 같습니다. Y로 조정할까요?"라고 구체적인 proposal을 내놓아야 해.

```python
class FailurePatternDetector:
    """Phase 3 반복 실패에서 패턴을 감지하고 적응 방안 제안"""
    
    def analyze(self, exec_history: list[ExecAttempt]) -> AdaptationProposal | None:
        errors = [a.error_type for a in exec_history]
        
        if errors.count("OOM") >= 3:
            return AdaptationProposal(
                type="downscale_model",
                description="GPU 메모리 부족이 반복됩니다.",
                options=[
                    "ViT-Tiny (85M → 5M params) + CIFAR-10으로 축소",
                    "ResNet-18 single model만 실행",
                    "batch_size 128 → 32로 축소 후 재시도",
                ]
            )
        
        if errors.count("TIMEOUT") >= 2:
            return AdaptationProposal(
                type="reduce_epochs",
                description="실행 타임아웃이 반복됩니다.",
                options=[
                    f"max_epoch를 {original_epochs} → {original_epochs // 3}로 축소",
                    "검증 주기 늘리기 (val_every_n_epochs=5)",
                ]
            )
        
        return None
```

**판정: 변환 적용.** 대화형 self-reframing은 제외. 대신 **Phase 3 실패 패턴 감지 + 계획 적응 제안** 형태로 적용. 검토 1의 `PlanAdaptationGate`와 통합.

---

### 통합 판정표

| GPT 조언 | 판정 | crewai_prototype 적용 형태 | 우선순위 |
|---------|------|--------------------------|---------|
| 1. State not Plan | 부분 변환 | Plan Adaptation Gate (Phase 3 반복 실패 시) | Tier 2 |
| 2. Multi-query retrieval | **제외** | 검색 인프라 없음. 미래 lit-search 단계에서 재검토 | — |
| 3. Implicit intent tracking | **제외** | 배치 파이프라인에서 안티패턴 (spec drift 유발) | — |
| 4. Context entropy / memory 계층 | **강력 적용** | 4계층 메모리 구조 (working/episodic/semantic/archival) → Phase 2 repair loop + Phase 경계 핸드오프 | **Tier 1** |
| 5. Tool arbitration | 부분 이미 구현 | Python orchestrator(policy) + ErrorClassifier(routing)는 기존 설계 포함. Uncertainty estimation만 신규 | Tier 3 |
| 6. 압축 > 생성 | **강력 적용** | ContextCompressor at Phase boundaries (특히 → Writer 입력) | **Tier 1** |
| 7. Self-reframing | 변환 적용 | FailurePatternDetector → AdaptationProposal (조언 1과 통합) | Tier 2 |

---

### 부록 결론

**Alex:**  
GPT 조언 7개 중 2개(3번 implicit intent, 2번 retrieval)는 우리 시스템에 적용하면 안 되는 패턴이야. 배치 파이프라인에 대화형 에이전트 설계를 억지로 적용하면 spec drift와 예측 불가능성이 올라가.

**Jordan:**  
반면 4번(context entropy)과 6번(압축 우선)은 기존 검토에서 다룬 문제들의 근본 원인을 잘 설명해주고 있어. 특히 GPT의 4계층 메모리 모델은 우리가 이미 감지한 "컨텍스트 오염" 문제에 대한 **설계 언어**를 제공해. 이걸 기반으로 Phase 2 repair loop의 프롬프트 구성을 재설계할 수 있어.

**Alex:**  
최종적으로 GPT 조언이 기존 검토에 추가하는 새로운 항목은 크게 두 가지야:

1. **4계층 메모리 구조** (working/episodic/semantic/archival) → Tier 1 기존 항목에 통합
2. **FailurePatternDetector + PlanAdaptationGate** → Tier 2 신규 항목으로 추가

이 두 개를 기존 우선순위 테이블에 반영하면 충분해.

---

*부록 작성일: 2026-05-27. GPT 조언 원문을 기반으로 crewai_prototype 코드와 비교 검토함.*

---

## 부록 2: 하이브리드 설계 검토 — "배치와 대화형의 적절한 혼합"

> **논의 배경**: 기존 batch 파이프라인 방향은 유지하되,  
> (1) 실행 전 불명확한 부분을 몇 가지 질문으로 확인하고,  
> (2) 실행 중 추가 정보를 주입할 수 있으며,  
> (3) 실행 후 추가 run 여부를 결정할 수 있는  
> **하이브리드 구조**로 발전시키는 방안을 검토한다.

---

### 전제 논의: 왜 지금 이 질문이 나오는가

---

**Jordan:**  
이 질문이 나온 배경을 먼저 짚어야 해. 지금까지 GPT 조언 검토에서 implicit intent tracking이나 evolving conversational state를 "배치 파이프라인에 안티패턴"이라고 결론 내렸는데, 그게 **완전한 거부**가 아니라 "잘못된 적용을 경계하라"는 의미였거든.

실제로 crewai_prototype을 직접 써보면 답답한 지점이 있어:
- "CIFAR-100 분류 비교" 한 줄만 주고 돌리면, 어떤 아키텍처를 비교할지 시스템이 자의적으로 결정해
- Phase 1이 잘못된 방향을 잡아도 ApprovalGate에서 전체 계획을 보고 "아, 이건 아닌데"라고 할 때는 이미 계획이 완성된 후야
- Phase 3이 끝났을 때 "accuracy 72%가 나왔는데 augmentation 추가하면 얼마나 오를지 궁금한데"라는 생각을 해도 그냥 끝나버려

**Alex:**  
이게 Codex에서도 제기됐던 질문이야. "완전 자율 실행"과 "사용자가 매번 확인"의 사이 어딘가에 최적점이 있어. 너무 자주 묻는 시스템은 자동화의 가치가 사라지고, 너무 적게 묻는 시스템은 사용자가 원하는 게 아닌 결과를 30분 기다린 후에 받아.

Codex 내부에서 이걸 **"autonomy dial"** 이라고 불렀어. 완전 자율(0)에서 완전 interactive(10) 사이에서, 연구용 파이프라인에서 최적점은 2~3 정도야.

---

### 1: 하이브리드 구조 개요 — 세 개의 대화 레이어

---

**Jordan:**  
하이브리드를 구현하기 전에, 어떤 종류의 대화가 필요한지 분류해야 해. 전부 동일한 "대화"가 아니거든.

```
레이어 1: Pre-flight Clarification (실행 전)
  - 언제: Phase 0 시작 전, 한 번만
  - 목적: 시스템이 합리적으로 추론할 수 없는 정보만 질문
  - 형태: 구조화된 질문 (선택지 있음), 최대 3~5개

레이어 2: In-run Context Injection (실행 중)
  - 언제: Phase 경계 (이미 있는 ApprovalGate/GuidanceGate 활용)
  - 목적: 사용자가 자발적으로 추가 정보를 주입
  - 형태: 자유 입력 → 시스템이 어떻게 반영할지 해석

레이어 3: Post-phase Extension Decision (단계 완료 후)
  - 언제: Phase 3 완료 후 (실험 결과가 나왔을 때)
  - 목적: 추가 run 여부 결정
  - 형태: 시스템이 제안 → 사용자가 승인/거부
```

**Alex:**  
이 세 레이어가 기존 파이프라인과 충돌하지 않으려면, **배치 파이프라인의 결정론적 실행을 건드리지 않는 인터페이스**로 설계해야 해. 즉, 레이어 1은 Phase 1 입력을 풍부하게 하고, 레이어 2는 Phase 경계 핸드오프에 context를 추가하고, 레이어 3은 완전히 새로운 run을 시작하거나 현재 run을 확장하는 거야.

핵심 원칙: **실행 중인 배치는 절대 수정하지 않는다.** 주입된 정보는 다음 Phase 또는 다음 run에 영향을 준다.

---

### 2: 레이어 1 — Pre-flight Clarification Protocol

---

**Alex:**  
이게 가장 설계하기 까다로운 부분이야. 뭘 물어볼지 결정하는 기준이 없으면, 너무 많이 묻거나 너무 적게 묻거나 둘 중 하나가 돼.

Codex에서 찾은 원칙은: **"LLM이 합리적인 default를 추론할 수 있는 것은 묻지 않는다."**

질문해야 하는 것:
1. 명시적 제약 (하드웨어, 시간, 예산) — LLM이 알 수 없음
2. 평가 기준 (어떤 metric을 최우선으로?) — 연구자마다 다름
3. 비교 대상 (어떤 아키텍처/방법론?) — 암묵적 선호가 있음
4. 허용된 라이브러리 — 환경 제약, LLM이 알 수 없음

묻지 않아야 하는 것:
- 배치 크기, 학습률 같은 하이퍼파라미터 — reasonable default 존재
- 코드 구조 — Designer가 결정
- 파일 명명 규칙 — 컨벤션으로 해결

**Jordan:**  
Claude Code에서도 slash command를 실행할 때 `/review`, `/test` 같은 명령에 인자가 없으면 현재 컨텍스트에서 추론해. 물어보는 건 정말 추론 불가능한 것만이야.

이걸 구현하는 방법으로 **Ambiguity Scorer**를 도입할 수 있어:

```python
class PreflightClarifier:
    """Phase 0 전에 최소한의 구조화된 질문을 생성한다."""
    
    MAX_QUESTIONS = 4  # 절대 넘지 않음
    
    def generate_questions(
        self, raw_topic: str, system_capabilities: SystemCapabilities
    ) -> list[ClarificationQuestion]:
        
        # LLM으로 어떤 부분이 모호한지 파악
        ambiguity_prompt = f"""
        Research topic: {raw_topic}
        
        Identify ONLY the information that:
        1. Cannot be reasonably inferred or defaulted
        2. Would significantly change the experiment design
        3. Is specific enough that a single short answer resolves it
        
        Output a JSON list of at most {self.MAX_QUESTIONS} questions.
        Each question must have:
        - "dimension": what aspect (hardware|objective|scope|constraint)
        - "question": the exact question text
        - "options": 2-4 concrete options (or null for open-ended)
        - "default": what the system will assume if skipped
        """
        
        questions_raw = self.llm.call([...])
        questions = parse_questions(questions_raw)
        
        # default가 없는 질문은 제외 (항상 진행할 수 있어야 함)
        return [q for q in questions if q.default is not None]
    
    def apply_answers(
        self,
        topic: str,
        answers: dict[str, str],
        questions: list[ClarificationQuestion],
    ) -> EnrichedTopic:
        """답변을 반영한 풍부한 연구 주제 반환"""
        # 미답변 → default 적용
        resolved = {
            q.dimension: answers.get(q.dimension, q.default)
            for q in questions
        }
        return EnrichedTopic(
            original=topic,
            constraints=resolved,
            # Phase 1 Planner에 전달될 추가 컨텍스트
            enriched_prompt=self._build_enriched_prompt(topic, resolved),
        )
```

**Alex:**  
중요한 설계 결정: **질문에 답하지 않아도 항상 진행할 수 있어야 해.** 모든 질문에는 `default`가 있고, 사용자가 타임아웃되거나 "skip"하면 default로 진행해. 사용자를 블로킹하는 pre-flight이 되면 안 돼.

구체적인 UX 흐름:
```
사용자: "CIFAR-100 분류 성능 비교 실험"

시스템:
  실험을 시작하기 전에 3가지 확인할게요. (스킵하면 기본값으로 진행)

  Q1. 비교할 아키텍처는?
      [ ] ResNet-18 vs ViT-Tiny        (기본값)
      [ ] ResNet-50 vs ViT-Base
      [ ] 커스텀 입력: ____________

  Q2. 가장 중요한 평가 지표는?
      [ ] Top-1 Accuracy               (기본값)
      [ ] Accuracy + 추론 속도
      [ ] Accuracy + 파라미터 수

  Q3. 사용 가능한 GPU 메모리 (GB)?
      [ ] 8GB 이하
      [ ] 16GB                         (기본값)
      [ ] 24GB 이상

  [60초 안에 응답하지 않으면 기본값으로 자동 시작]
```

**Jordan:**  
그리고 이 질문들은 Phase 1 Planner에게 **직접** 전달돼야 해. 현재 Planner 프롬프트가 연구 주제만 받는데, 이 구조화된 제약이 추가되면 Designer의 파일 구조 결정에도 영향을 줄 수 있어.

---

### 3: 레이어 2 — In-run Context Injection

---

**Jordan:**  
현재 코드에는 ApprovalGate와 GuidanceGate가 이미 존재해:
- `ApprovalGate`: Phase 1 계획을 사용자가 승인
- `GuidanceGate`: Phase 2/3 repair 실패 시 사용자에게 힌트 요청

이게 실은 이미 "in-run interaction"이야. 문제는 이 gate들이 **blocking**이고 **수동적**이라는 거야 — 시스템이 막혔을 때만 사용자에게 물어봐.

하이브리드에서 추가할 것은 **사용자가 자발적으로** context를 주입할 수 있는 채널이야. 실행이 잘 진행 중인데도 "아, 이 실험에서 dropout을 추가해보면 어떨까"라는 생각이 들면 지금은 아무 방법이 없어.

**Alex:**  
이걸 구현하는 가장 안전한 방식은 **"staged injection"** — 주입된 정보가 즉시 실행 중인 코드에 반영되는 게 아니라, 다음 Phase 시작 시점에 적용되는 거야.

```python
class ContextInjectionQueue:
    """
    사용자가 실행 중 주입하는 context를 큐에 쌓고,
    다음 Phase 경계에서 소비한다.
    """
    
    def inject(self, user_message: str, current_phase: str):
        entry = InjectionEntry(
            message=user_message,
            injected_at_phase=current_phase,
            timestamp=time.time(),
            status="pending",
        )
        self._queue.append(entry)
        emit("CONTEXT_INJECTED", {
            "message": user_message,
            "will_apply_at": self._next_phase(current_phase),
        })
    
    def consume_for_phase(self, phase: str) -> list[InjectionEntry]:
        """Phase 시작 시 큐에서 해당 Phase 이전에 주입된 항목 소비"""
        applicable = [e for e in self._queue if e.status == "pending"]
        for e in applicable:
            e.status = "consumed"
            e.consumed_at_phase = phase
        return applicable
```

**Jordan:**  
그리고 주입된 정보를 **어떻게 해석할지**가 핵심이야. 사용자가 "dropout 추가해봐"라고 했을 때, 이게:
- Phase 2 Coder에게 코드 수정을 요청하는 건지
- Phase 1 Designer에게 파일 구조를 바꾸라는 건지  
- 현재 run은 유지하고 다음 run에서 변형을 시도하라는 건지

이를 LLM이 해석해서 분류해야 해:

```python
class InjectionInterpreter:
    
    def interpret(
        self,
        message: str,
        current_phase: str,
        plan_bundle: PlanBundle,
    ) -> InjectionIntent:
        
        prompt = f"""
        Current pipeline phase: {current_phase}
        User injected: "{message}"
        Current plan summary: {plan_bundle.brief_summary()}
        
        Classify this injection:
        A) MODIFY_CURRENT_RUN — change something in remaining phases (what? how?)
        B) QUEUE_NEXT_RUN — run a variant after current completes (describe variant)
        C) INFORMATIONAL — add context but no action needed
        D) CLARIFICATION_NEEDED — ambiguous, ask user to be specific
        
        Output JSON: {{ "type": "A|B|C|D", "action": "...", "confidence": 0.0-1.0 }}
        """
        
        result = self.llm.call([...])
        intent = parse_injection_intent(result)
        
        if intent.confidence < 0.7:
            intent.type = "CLARIFICATION_NEEDED"
        
        return intent
```

**Alex:**  
한 가지 강한 제약을 둬야 해: **Phase 2가 실행 중일 때는 코드 구조 변경을 받지 않는다.** Phase 2가 파일을 생성 중인데 "아 모델 구조를 바꿔줘"라는 주입이 들어오면, 생성된 파일 10개는 어쩔 건데? 이게 파이프라인 불일치의 가장 위험한 케이스야.

안전한 주입 가능 시점:

```
Phase 0 완료 후 → Phase 1 시작 전   : 연구 제약 추가 가능 (A형)
Phase 1 완료 후 → Phase 2 시작 전   : 설계 수정 가능 (A형) + ApprovalGate에 통합
Phase 2 실행 중                      : 큐에만 저장, 즉시 적용 금지 (B형만)
Phase 3 완료 후 → Phase 4 시작 전   : 분석 방향 추가 가능 (A형)
Phase 4 완료 후                      : 다음 run 제안만 가능 (B형)
```

---

### 4: 레이어 3 — Post-phase Extension Decision

---

**Alex:**  
이게 가장 흥미롭고 연구적으로도 가장 가치 있는 레이어야. 첫 번째 run이 완료됐을 때 "여기서 끝낼 것인가, 변형 실험을 더 할 것인가"를 시스템이 제안해주는 거야.

현재 코드는 Phase 4가 완료되면 그냥 끝나. 연구자 입장에서 "72% accuracy가 나왔는데 data augmentation을 추가하면 어떨까?"라는 생각은 Phase 4 보고서를 다 읽고 나서야 드는데, 그걸 다음 run으로 이어주는 메커니즘이 없어.

**Jordan:**  
이걸 **"Research Extension Proposer"**로 구현할 수 있어. Phase 3 결과와 Phase 1 계획을 분석해서 자연스러운 다음 실험을 제안하는 거야:

```python
class ResearchExtensionProposer:
    """
    현재 run의 결과를 분석해서 가치 있는 후속 실험을 제안한다.
    """
    
    def propose(
        self,
        plan: PlannerResult,
        exec_result: ExecutorResult,
        writing_result: WritingResult,
    ) -> list[ExtensionProposal]:
        
        prompt = f"""
        Completed experiment: {plan.objective}
        Key results: {exec_result.metrics}
        
        Based on these results, propose 2-3 natural follow-up experiments.
        Each proposal must:
        1. Be a concrete, runnable variant (not vague)
        2. Address an open question raised by the current results
        3. Estimate additional compute time
        4. State what new insight it would provide
        
        Output JSON array of proposals.
        """
        
        raw = self.llm.call([...])
        proposals = parse_proposals(raw)
        
        # 예상 시간 정렬 (짧은 것 먼저)
        return sorted(proposals, key=lambda p: p.estimated_minutes)
```

**Alex:**  
그리고 이 제안을 사용자에게 어떻게 보여줄지가 UX의 핵심이야. 너무 많은 선택지는 피로도를 높여:

```
Phase 4 완료. 실험 결과: ResNet-18 72.1%, ViT-Tiny 69.4% on CIFAR-100

후속 실험 제안 (선택 또는 스킵):

  [A] Data Augmentation 추가 (예상 +15분)
      → CutMix + AutoAugment 적용, 예상 +2~3%p 향상 가능

  [B] 더 큰 모델 비교 (예상 +40분)
      → ResNet-50 vs ViT-Small로 확장, 성능 상한 파악

  [C] 학습률 스케줄러 비교 (예상 +20분)
      → CosineAnnealing vs StepLR, ViT 성능 차이 분석

  [D] 이 결과로 보고서 완료

  [타임아웃 60초 → D 자동 선택]
```

**Jordan:**  
여기서 중요한 설계 결정이 하나 있어 — **"수정된 현재 run"인가 "완전히 새로운 run"인가.**

- 옵션 A(Data Augmentation): Phase 1 설계를 약간 수정 → Phase 2부터 재실행이 효율적
- 옵션 B(더 큰 모델): Phase 1 설계가 크게 바뀜 → 완전 새 run
- 옵션 C(학습률 스케줄러): Phase 2 일부 파일만 수정 → Phase 2 부분 재실행

이걸 자동으로 판단해서 **최소한의 재실행**으로 처리할 수 있으면 시간을 크게 절약해. Phase 2 코드 일부만 바꾸면 되는데 전체를 다시 돌리는 건 낭비야.

```python
class ExtensionRouter:
    
    def route(
        self,
        proposal: ExtensionProposal,
        existing_checkpoint: RunCheckpoint,
    ) -> ExtensionRoute:
        
        if proposal.modifies_phase1_design:
            # Phase 1 재실행 (거의 처음부터)
            return ExtensionRoute(restart_from="phase1", reuse_workspace=False)
        
        if proposal.modifies_files <= 3:
            # 특정 파일만 재생성 후 Phase 3부터 재실행
            return ExtensionRoute(
                restart_from="phase3",
                reuse_workspace=True,
                files_to_regenerate=proposal.affected_files,
            )
        
        # 그 외: Phase 2부터 재실행 (workspace 재사용)
        return ExtensionRoute(restart_from="phase2", reuse_workspace=True)
```

---

### 5: 안티패턴 — "하이브리드"가 잘못될 때

---

**Jordan:**  
하이브리드 설계에서 가장 흔한 실패 패턴을 명시적으로 정의해야 해. 구현하다 보면 이 방향으로 흘러가기 쉬워.

**안티패턴 1: "질문 폭격"**  
Pre-flight에서 10개 이상 질문. 사용자가 이걸 전부 답하려면 오히려 더 오래 걸려. LLM이 reasonable default를 추론할 수 있는 것들까지 묻게 됨.

**대응**: MAX_QUESTIONS = 4 하드캡. "LLM이 default를 가질 수 없는 것"만 질문.

**Alex:**  
**안티패턴 2: "실행 중 코드 변경"**  
Phase 2가 진행 중일 때 사용자 주입을 즉시 반영하려다 파이프라인 불일치 발생. 생성된 파일 10개는 이전 spec, 남은 파일 5개는 새 spec으로 만들어지면 import 충돌이 일어나.

**대응**: 실행 중 주입은 큐에만 저장, 다음 Phase 경계에서만 소비.

**Jordan:**  
**안티패턴 3: "Extension 지옥"**  
후속 실험 제안이 계속 나와서 실험이 끝나지 않는 상태. A를 하면 B가 제안되고, B를 하면 C가 제안되고...

**대응**: Extension은 최대 2단계만 허용. 원본 run에서 1단계 extension, extension run에서 1단계만 추가 허용. 그 이상은 새 run으로 분리.

**Alex:**  
**안티패턴 4: "대화형 Drift"**  
사용자 주입이 쌓이면서 Phase 1의 원래 연구 계획과 멀어지는 상태. "CIFAR-100 비교"로 시작했는데 5번 주입 후 "NLP 태스크 비교"로 바뀌어 있는 경우.

**대응**: `InjectionInterpreter`가 "이 주입이 원래 연구 목적과 상충하는가"를 체크. 상충하면 경고 표시 또는 새 run 권장.

---

### 6: 현재 코드와의 통합 지점

---

**Jordan:**  
하이브리드 구조가 현재 crewai_prototype 코드에 어떻게 통합되는지 매핑해보자:

| 레이어 | 현재 코드 통합 지점 | 변경 범위 |
|--------|------------------|---------|
| Pre-flight Clarification | `main.py` CLI 진입점 전 단계 추가 | 신규 클래스 (`PreflightClarifier`) + API endpoint 1개 |
| In-run Injection Queue | `orchestration/` 내 `ContextInjectionQueue` 추가 | `phases/*.py`의 Phase 시작 부분에서 큐 소비 |
| ApprovalGate 강화 | `orchestration/approval_registry.py` 확장 | 기존 Gate에 주입 소비 로직 추가 |
| Extension Proposer | Phase 4 완료 후 신규 단계 | `phase5_extension.py` (새 파일) 또는 `main.py` 후처리 |
| Extension Router | `orchestration/` 신규 | 체크포인트 시스템과 결합 (1부 Tier 1) |

**Alex:**  
특히 Extension Router는 체크포인트 시스템 없이는 구현할 수 없어. Phase 3부터 재실행하려면 Phase 2 결과가 저장되어 있어야 하거든. 1부에서 제안한 `CheckpointManager` (Tier 1)가 선행 조건이야.

---

### 7: 설계 결론 — 무엇을 구현할 것인가

---

**Alex:**  
하이브리드 3개 레이어의 구현 순서를 명확히 해야 해. 동시에 다 할 수는 없어.

**Jordan:**  
우선순위를 기술적 의존성과 연구 가치 기준으로 정리하면:

```
Step 1: CheckpointManager 구현 (기존 Tier 1)
  ↓ (선행 조건)
Step 2: Pre-flight Clarification (레이어 1)
  - 진입 장벽 낮음, 연구 결과 품질 즉시 향상
  - Phase 1 Planner에 EnrichedTopic 전달 구조 추가만 필요
  - 독립적으로 구현 가능, 체크포인트 불필요
  ↓
Step 3: In-run Context Injection Queue (레이어 2)
  - CheckpointManager와 결합해야 안전
  - ApprovalGate 강화로 구현
  ↓
Step 4: Extension Proposer + Router (레이어 3)
  - CheckpointManager + Injection Queue 모두 필요
  - 가장 복잡하지만 연구 비교 단계에서 가장 가치 있음
```

**Alex:**  
그리고 이 하이브리드 구조가 AutoGen/LangGraph 비교에서 어떤 의미를 갖는지도 생각해봐야 해. 세 프레임워크가 동일한 하이브리드 인터페이스를 갖추면, "사용자 개입 필요 횟수", "extension 성공률" 같은 새로운 비교 차원이 생겨. 그냥 "accuracy" 비교보다 훨씬 풍부한 프레임워크 비교가 될 거야.

**Jordan:**  
최종 정리:

| 레이어 | 구현 복잡도 | 연구 가치 | 의존성 |
|--------|-----------|---------|--------|
| Pre-flight Clarification | 낮음 | 높음 (즉각) | 없음 |
| In-run Injection | 중간 | 중간 | CheckpointManager |
| Extension Proposer/Router | 높음 | 높음 (비교 연구) | 1+2 모두 |

**Pre-flight Clarification을 먼저 구현하는 것이 가장 가성비가 높다.** 체크포인트 없이도 독립적으로 동작하고, 연구 계획의 품질을 바로 높이며, 사용자 경험을 즉각 개선한다. Extension 시스템은 세 프레임워크가 모두 안정화된 이후 비교 연구 단계에서 구현하면 된다.

---

*부록 2 작성일: 2026-05-27. 기존 batch 파이프라인 방향을 유지하면서 대화형 요소를 구조적으로 통합하는 하이브리드 설계 방안을 검토함.*
