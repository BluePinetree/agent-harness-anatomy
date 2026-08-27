# MARS 버그 수정 패치 노트 — 2026년 6월 세션

> 이 문서는 2026년 6월 개발 세션에서 발견·수정된 버그를 기록한다.  
> 각 항목은 증상 → 근본 원인 → 수정 위치 → 핵심 코드 순으로 구성된다.  
> 관련 ADR: [ADR-008](decisions/ADR-008-repair-loop-escalation.md), [ADR-013](decisions/ADR-013-phase3-success-criterion-rc-vs-result-json.md), [insight: success_signal_reliability](insights/success_signal_reliability.md)

---

## [2026-06-17] dep context 예산 초과 → 의존 파일 무음 드롭 → cross-module 함수명 불일치

### 증상

Phase 3 실행 시 `ImportError: cannot import name 'measure_inference_latency' from 'bench.inference_bench'`.  
실제 export 이름은 `benchmark_inference_latency`였지만 LLM이 잘못된 이름을 사용해 코드를 생성했다.

### 근본 원인

`_build_dep_context()`가 의존 파일을 전체 내용 그대로 누적하다가  
`_MAX_TOTAL_DEP_CHARS = 18,000` 예산 초과 시 이후 파일을 **무음으로 드롭**한다.  
LLM은 해당 파일의 실제 시그니처를 보지 못하고 이름을 추측하게 된다.

### 수정 위치

`crewai_prototype/phases/phase2_coding.py` — `_extract_api_surface()` / `_build_dep_context()`

### 핵심 코드

```python
def _extract_api_surface(source: str) -> str:
    """AST로 API 표면만 추출한다: import + __all__ + 함수/클래스 시그니처 + 첫 docstring.
    구현 본문 제거 → 평균 80% 크기 절감.
    """
    try:
        tree = ast.parse(source)
    except SyntaxError:
        return source[:_MAX_DEP_CHARS]

    lines = source.splitlines()
    kept: list[str] = []

    for node in ast.walk(tree):
        # import 문 전체 보존
        if isinstance(node, (ast.Import, ast.ImportFrom)):
            kept.append(ast.unparse(node))
        # __all__ 보존
        elif (isinstance(node, ast.Assign)
              and any(isinstance(t, ast.Name) and t.id == "__all__" for t in node.targets)):
            kept.append(ast.unparse(node))
        # 함수/클래스: 시그니처 + 첫 docstring만, 본문 제거
        elif isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef)):
            sig = ast.unparse(node).split("\n")[0]          # def/class 줄
            kept.append(sig)
            # 첫 docstring
            if (node.body and isinstance(node.body[0], ast.Expr)
                    and isinstance(node.body[0].value, ast.Constant)
                    and isinstance(node.body[0].value.value, str)):
                doc = node.body[0].value.value.strip().split("\n")[0]
                kept.append(f'    """{doc}"""')

    return "\n".join(kept)[:_MAX_DEP_CHARS]
```

### 결과

8개 이상의 dep 파일이 있어도 전부 예산 내에 포함됨. cross-module 함수명 불일치 재발 없음.

---

## [2026-06-19] 서브디렉토리 패키지 `__init__.py` 누락 + PYTHONPATH 미설정 → ModuleNotFoundError

### 증상

Phase 3에서 `ModuleNotFoundError: No module named 'bench'`.  
LLM이 `src/bench/inference_bench.py`를 생성했으나 `src/bench/__init__.py`가 없고,  
`subprocess.Popen`의 환경에 `src/`가 `PYTHONPATH`에 없었다.

### 근본 원인

- `_write_to_disk()`가 `.py` 파일을 쓸 때 상위 디렉토리에 `__init__.py`를 자동 생성하지 않음.
- Phase 3 `_run_script()`의 `subprocess.Popen` 환경에 `PYTHONPATH`를 주입하지 않음.

### 수정 위치

- `crewai_prototype/phases/phase2_coding.py` — `_ensure_init_files()` 추가 및 `_write_to_disk()`에서 호출
- `crewai_prototype/phases/phase3_execution.py` — `_run_script()` `Popen` 환경변수에 PYTHONPATH 주입

### 핵심 코드

```python
# phase2_coding.py
def _ensure_init_files(directory: Path, workspace_root: Path) -> None:
    """directory부터 workspace_root/src까지 모든 상위에 __init__.py 생성."""
    src_root = workspace_root / "src"
    current = directory
    while current != src_root and src_root in current.parents or current == src_root:
        init = current / "__init__.py"
        if not init.exists():
            init.write_text("", encoding="utf-8")
        if current == src_root:
            break
        current = current.parent

# phase3_execution.py — _run_script()
env = os.environ.copy()
src_dir = str(Path(workspace_root) / "src")
existing_path = env.get("PYTHONPATH", "")
env["PYTHONPATH"] = f"{src_dir}{os.pathsep}{existing_path}" if existing_path else src_dir

data_dir = DATA_CACHE_DIR or str(Path.home() / ".cache" / "mars_datasets")
Path(data_dir).mkdir(parents=True, exist_ok=True)
env["DATA_DIR"] = data_dir
```

### 결과

`from bench.xxx import ...` 형태의 절대 import가 Phase 3에서 정상 동작.  
`DATA_DIR` 주입으로 데이터셋 영구 캐시도 함께 해결.

---

## [2026-06-22] 데이터셋 다운로드 timeout → Analyzer가 `repair_files:[]` 반환 → 무한 retry 낭비

### 증상

CIFAR-10 다운로드가 5400초(90분) timeout으로 실패.  
Analyzer가 환경 문제를 인식하고 `"repair_files": []`를 반환했으나,  
파이프라인이 이를 "수정 불가" 신호로 처리하지 않고 MAX_EXEC_REPAIR_ATTEMPTS(3)까지 retry를 반복.  
총 4.5시간 낭비.

### 근본 원인

에스컬레이션 조건이 `attempt >= MAX_EXEC_REPAIR_ATTEMPTS`만 체크했음.  
`repair_files`가 빈 경우 (환경/네트워크 문제 → 코드 수정으로 해결 불가)는 즉시 에스컬레이션해야 하지만 이 분기가 없었다.

### 수정 위치

`crewai_prototype/phases/phase3_execution.py` — `run_execution_phase()` 에스컬레이션 조건

### 핵심 코드

```python
# Before
if attempt >= MAX_EXEC_REPAIR_ATTEMPTS:
    # escalate

# After
if attempt >= MAX_EXEC_REPAIR_ATTEMPTS or not repair_files:
    reason = (
        f"after {attempt} attempts"
        if attempt >= MAX_EXEC_REPAIR_ATTEMPTS
        else "no fixable files identified (likely environment/network issue)"
    )
    # escalate immediately
```

### 결과

네트워크/환경 오류 시 첫 번째 실패 후 즉시 GuidanceDrawer 팝업.  
불필요한 retry로 인한 시간 낭비 제거.

---

## [2026-06-23] Stage 3 코드가 Stage 1 dataclass에 없는 필드 사용 → 반복 TypeError

### 증상

`TypeError: RunConfig.__init__() got an unexpected keyword argument 'amp'`  
`TypeError: DataConfig.__init__() got an unexpected keyword argument 'augment'`  
같은 패턴의 런타임 TypeError가 실험 실행마다 반복됨. repair가 돼도 다시 발생.

### 근본 원인 (3가지 결함의 복합)

1. **Stage 3 dep 누락**: `run_coding_phase()`가 Stage 3 파일의 `imports_from`을 Designer 출력 그대로 사용. Designer가 Stage 1 파일을 누락하면 LLM은 dataclass 정의를 보지 못하고 필드를 추측한다.

2. **`check_import()` 스킵**: `_SKIP_IMPORT_PATTERNS`가 `torch`를 import하는 파일을 무조건 Pass 처리. `src/main.py`는 항상 torch를 import하므로 실질적으로 import check가 작동하지 않았다.

3. **repair task에 Stage 1 API 없음**: Phase 3 수정 에이전트가 Stage 1 클래스 정의를 받지 못해 잘못된 필드를 반복해서 추측.

### 수정 위치

- `crewai_prototype/phases/phase2_coding.py` — Stage 3 루프에서 `imports_from` 강제 보강
- `crewai_prototype/crew_tools/syntax_check_tool.py` — `check_dataclass_fields()` 추가
- `crewai_prototype/phases/phase3_execution.py` — `_EXEC_REPAIR_TASK`에 `{stage1_api}` 주입

### 핵심 코드

```python
# 1. phase2_coding.py — Stage 3 루프에서 Stage 1 파일을 imports_from에 강제 추가
if stage_num == 3:
    stage1_written = [
        fr.path
        for s in coding_result.stages
        if s.stage == 1
        for fr in s.files
        if fr.written
    ]
    extra = [p for p in stage1_written if p not in file_spec.imports_from]
    if extra:
        file_spec = file_spec.model_copy(
            update={"imports_from": file_spec.imports_from + extra}
        )

# 2. syntax_check_tool.py — AST 기반 dataclass 필드 검증
def check_dataclass_fields(entry_path, workspace_root):
    # 워크스페이스 전체에서 @dataclass 정의 수집
    dataclass_fields: dict[str, set[str]] = {}
    for py_file in workspace.rglob("*.py"):
        tree = ast.parse(py_file.read_text(...))
        for node in ast.walk(tree):
            if isinstance(node, ast.ClassDef) and is_dataclass(node):
                dataclass_fields[node.name] = {field.target.id for field in ...}

    # entry point 호출부의 kwarg와 대조
    for node in ast.walk(entry_tree):
        if isinstance(node, ast.Call) and class_name in dataclass_fields:
            for kw in node.keywords:
                if kw.arg not in dataclass_fields[class_name]:
                    return CheckResult(passed=False, error=f"TypeError: {class_name}() got unexpected keyword argument '{kw.arg}'")

# 3. phase3_execution.py — repair task에 Stage 1 API 주입
stage1_paths = [fr.path for s in coding_result.stages if s.stage == 1 for fr in s.files if fr.written]
stage1_api = _build_dep_context(stage1_paths, workspace_root) if stage1_paths else "(none)"

# _EXEC_REPAIR_TASK 프롬프트에 포함:
# "Stage 1 API definitions (authoritative — these are the ONLY valid signatures):\n{stage1_api}"
```

### 결과

- Stage 3 생성 시 Stage 1 클래스 정의가 프롬프트에 반드시 포함됨
- `check_dataclass_fields()`가 Phase 2 체크 단계에서 잘못된 kwarg를 조기 탐지
- repair 에이전트도 올바른 클래스 시그니처를 보고 수정

---

## [2026-06-23] Success Signal Reliability Gap — rc=0이어도 result.json.success=False면 성공으로 처리

### 증상

ML 스크립트가 `TypeError: RunConfig.__init__() got unexpected keyword argument 'name'`로 실패했음에도  
UI에 "추가 실험 제안" 화면이 출력됐다.  
`result.json`에 `success=False`가 명시돼 있었지만 파이프라인은 이를 무시하고 Phase 4로 진행했다.

### 근본 원인

`run_execution_phase()`가 `return_code == 0`을 유일한 성공 기준으로 사용.  
`result.json.success=False`는 경고 메시지만 emit하고 실패 처리를 하지 않았다.

ML 스크립트는 부분 결과 보존을 위해 일반적으로 `try/except`로 예외를 잡고 graceful shutdown한다:

```python
try:
    run_experiment()
except Exception as e:
    save_results({"error": str(e), "success": False})
    # sys.exit(1) 없음 → rc=0
```

이 패턴은 Specification Gaming / Success Signal Reliability Gap으로 불리는 Agentic AI의 알려진 문제.  
자세한 자료조사: [docs/insights/success_signal_reliability.md](insights/success_signal_reliability.md)

### 수정 위치

`crewai_prototype/phases/phase3_execution.py` — `run_execution_phase()` rc==0 분기

### 핵심 코드

```python
# Defense in Depth — 3계층 검증
if run_result["return_code"] == 0:
    rj = run_result.get("result_json", {})
    metrics = rj if isinstance(rj, dict) else {}

    # L2: result.json.success=False → 실패 경로
    if not metrics.get("success", True):
        error_msg = str(metrics.get("error", "result.json.success=False"))[:400]
        emit("AGENT_MESSAGE", f"[Phase 3] rc=0 but result.json.success=False — treating as failure: {error_msg}", ...)
        run_result = {**run_result, "return_code": -3, "stderr_tail": error_msg}
        # → 아래 실패 경로로 낙하
    else:
        # L3: numeric metric 없으면 advisory 경고 (실패 처리는 하지 않음)
        has_numeric = any(isinstance(v, (int, float)) for k, v in metrics.items() if k != "success")
        if not has_numeric:
            emit("AGENT_MESSAGE", "[Phase 3] Warning: result.json has no numeric metrics.", ...)
        return ExecutorResult(success=True, ...)

# 실패 경로 (rc != 0 또는 L2 실패)
stderr = run_result["stderr_tail"]
...
```

| 계층 | 신호 | 실패 시 처리 |
|------|------|------------|
| L1 | `return_code == 0` | 실패 경로 (분석·수정·재시도) |
| L2 | `result.json.success == True` | 실패 경로 (rc=0이어도) ← 이번 수정 |
| L3 | 숫자 metric 1개 이상 | 경고 (advisory) |

### 결과

`success=False`가 담긴 `result.json`이 있을 때 파이프라인이 실패 경로로 분기하고  
Phase 4 진행 없이 repair → escalation 흐름을 탄다. 위양성 완료(False Positive Completion) 제거.

---

## [2026-06-23] ApprovalDialog 너비 고정 — shadcn DialogContent `sm:max-w-lg` 하드코딩 충돌

### 증상

ApprovalDialog가 브라우저 창 크기와 무관하게 좁게 표시됨 (약 512px 고정).  
`w-[96vw] max-w-5xl` 클래스를 추가해도 효과 없음.

### 근본 원인

`research_system_ui/client/src/components/ui/dialog.tsx`의 `DialogContent` 기본 클래스에  
`sm:max-w-lg`가 하드코딩돼 있다. Tailwind CSS 충돌 해소 방식 때문에 `max-w-5xl` 같은  
추가 클래스가 `sm:max-w-lg`를 덮어쓰지 못한다.

### 수정 위치

`research_system_ui/client/src/components/ApprovalDialog.tsx` — `DialogContent` className

### 핵심 코드

```tsx
// Before
<DialogContent className="w-[96vw] max-w-5xl h-[90vh] ...">

// After — max-w-[90vw]로 sm:max-w-lg를 명시적으로 override
<DialogContent className="w-[90vw] max-w-[90vw] h-[90vh] flex flex-col gap-0 p-0 overflow-hidden rounded-xl border-0 shadow-2xl [&>button]:hidden">
```

`max-w-[90vw]` (임의값 클래스)는 shadcn의 `sm:max-w-lg` (미리 정의 클래스)보다 specificity 우선순위가 높다.

### 결과

대화창이 브라우저 뷰포트의 90% 너비를 유지하며 반응형으로 동작.

---

---

## [2026-06-24] 인스턴스 속성 접근 미탐지 — `spec.aug` AttributeError Phase 2 통과

### 증상

Phase 3에서 `AttributeError: 'ExperimentSpec' object has no attribute 'aug'`.  
Phase 2의 `check_dataclass_fields()`를 통과했으나 실행 시 크래시.

### 근본 원인

`check_dataclass_fields()`가 `ClassName(kwarg=...)` 생성자 호출(AST `Call` 노드)만 검사하고  
`spec.aug` 같은 인스턴스 속성 접근(AST `Attribute` 노드)은 검사하지 않았다.

| 패턴 | AST 노드 | 기존 커버 |
|------|---------|---------|
| `RunConfig(amp=True)` | `ast.Call` + keywords | ✅ |
| `spec.aug` | `ast.Attribute` | ❌ → 이번 수정 |

### 수정 위치

`crewai_prototype/crew_tools/syntax_check_tool.py` — `check_dataclass_fields()`

### 핵심 코드

```python
# Step 2: varname = ClassName(...) 직접 대입 추적 → {varname: ClassName} 매핑
var_types: dict[str, str] = {}
for node in ast.walk(entry_tree):
    if isinstance(node, ast.Assign):
        if (len(node.targets) == 1
                and isinstance(node.targets[0], ast.Name)
                and isinstance(node.value, ast.Call)):
            varname = node.targets[0].id
            call = node.value
            if isinstance(call.func, ast.Name) and call.func.id in dataclass_fields:
                var_types[varname] = call.func.id

# Step 4: varname.attr 접근 검증
for node in ast.walk(entry_tree):
    if not isinstance(node, ast.Attribute):
        continue
    if not isinstance(node.value, ast.Name):
        continue
    varname = node.value.id
    if varname not in var_types:
        continue
    class_name = var_types[varname]
    if node.attr not in dataclass_fields[class_name]:
        return CheckResult(
            passed=False,
            error=f"AttributeError: '{class_name}' object has no attribute '{node.attr}' "
                  f"(accessed as {varname}.{node.attr}). Valid fields: {sorted(dataclass_fields[class_name])}",
            error_type="runtime",
            line_no=getattr(node, "lineno", None),
        )
```

### 한계

`varname = get_spec()` 처럼 함수 반환값으로 할당되는 경우는 추적하지 않는다.  
직접 대입(`varname = ClassName(...)`)만 커버한다.

---

## [2026-06-24] Windows `UnicodeDecodeError: cp949` — subprocess stdout 디코딩 실패

### 증상

Phase 3 실행 중 백그라운드 스트리밍 스레드가 `UnicodeDecodeError: 'cp949' codec can't decode byte` 로 크래시.  
자가수리 루프가 3회 돌았으나 생성 코드를 아무리 고쳐도 해결되지 않았다.

### 근본 원인

`subprocess.Popen(text=True)`는 시스템 기본 인코딩으로 stdout을 디코딩한다.  
한국어 Windows의 기본 인코딩은 **cp949**이며, 자식 프로세스가 출력하는 UTF-8 바이트 중 cp949에서 유효하지 않은 시퀀스가 있으면 `UnicodeDecodeError`가 발생한다.

이 버그는 **오케스트레이션 코드**(`phase3_execution.py`)에 있으므로 repair agent의 수리 범위(workspace 생성 코드) 밖이다. 자가수리가 구조적으로 불가능한 이유이기도 하다.

### 수정 위치

`crewai_prototype/phases/phase3_execution.py` — `_run_script()` `subprocess.Popen`

### 핵심 코드

```python
# Before
proc = subprocess.Popen(
    cmd, cwd=workspace_root, env=env,
    stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    text=True,
)

# After
proc = subprocess.Popen(
    cmd, cwd=workspace_root, env=env,
    stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    text=True,
    encoding="utf-8",
    errors="replace",   # 디코딩 불가 바이트 → ? 치환, 크래시 없음
)
```

---

## 수정 파일 요약

| 파일 | 수정 내용 |
|------|-----------|
| `crewai_prototype/phases/phase2_coding.py` | `_extract_api_surface()`, `_ensure_init_files()`, Stage 3 `imports_from` 강제 보강 |
| `crewai_prototype/phases/phase3_execution.py` | PYTHONPATH/DATA_DIR 주입, 에스컬레이션 조건 수정, L2/L3 성공 검증, Stage 1 API repair 주입, **cp949 인코딩 수정** |
| `crewai_prototype/crew_tools/syntax_check_tool.py` | `check_dataclass_fields()` AST 검사 추가, **인스턴스 속성 접근 검증 추가** |
| `crewai_prototype/pipeline_config/constants.py` | `DATA_CACHE_DIR` 상수 추가 |
| `research_system_ui/client/src/components/ApprovalDialog.tsx` | DialogContent 너비 viewport 비율로 수정 |

## 관련 문서

- [docs/insights/success_signal_reliability.md](insights/success_signal_reliability.md) — Success Signal Reliability Gap 자료조사 및 Solution Insight
- [docs/decisions/ADR-013-phase3-success-criterion-rc-vs-result-json.md](decisions/ADR-013-phase3-success-criterion-rc-vs-result-json.md)
- [docs/decisions/ADR-008-repair-loop-escalation.md](decisions/ADR-008-repair-loop-escalation.md)

---

## 참고 문헌

### Success Signal Reliability Gap (06-23 수정 관련)

1. Krakovna, V., Uesato, J., Mikulik, V., Martic, M., Tomasev, N., Stepleton, T., Hadfied-Menell, D., & Leike, J. (2020). **Specification gaming: the flip side of AI ingenuity.** DeepMind Blog.  
   — rc=0을 유일한 성공 기준으로 사용하는 것이 "Specification Gaming"임을 체계화한 최초 논문. 100개 이상의 사례 카탈로그 포함.

2. Lightman, H., Kosaraju, V., Burda, Y., Edwards, H., Baker, B., Lee, T., Leike, J., Schulman, J., Sutskever, I., & Cobbe, K. (2023). **Let's Verify Step by Step.** OpenAI. arXiv:2305.20050.  
   — Outcome Supervision(최종 결과만 검증) vs Process Supervision(단계별 검증)의 신뢰성 차이를 실증. **단, 본 논문의 Process Supervision은 훈련된 Reward Model이 각 추론 단계의 정오를 판별하는 구조로, 우리가 구현한 L2/L3 outcome-level 체크와 구현 방식이 다르다.** "단일 신호보다 다층 검증이 신뢰성이 높다"는 원칙만 참조했으며, 에폭별 loss 모니터링 등 진정한 Process Supervision은 미구현 상태(중기 과제).

3. Kinniment, M., Sato, L. K., Du, H., Goodrich, B., Hasin, M., Chan, L., Miles, B., Lin, T., Wijk, H., Kaufmann, M., Ho, M., Reuel, A., & Barnes, E. (2023). **Evaluating Language-Model Agents on Realistic Autonomous Tasks.** ARC Evals. arXiv:2312.11671.  
   — 코드 실행 에이전트에서 graceful shutdown 패턴이 오케스트레이터에게 성공 신호 오염을 일으킨다는 점을 명시.

4. Liu, X., Yu, H., Zhang, H., Xu, Y., Lei, X., Lai, H., Gu, Y., Ding, H., Men, K., Yang, K., Zhang, S., Deng, X., Zeng, A., Du, Z., Zhang, C., Shen, S., Zhang, T., Su, Y., … Tang, J. (2023). **AgentBench: Evaluating LLMs as Agents.** arXiv:2308.03688.  
   — exit code + 출력 파일 존재 + 내용 검증의 3-layer 검증 방식을 AgentBench 표준으로 채택. L1/L2/L3 검증 구조의 직접적 선례.

5. Krakovna, V. (2018, 지속 업데이트). **Specification gaming examples in AI.** GitHub compendium.  
   — Specification Gaming 사례 공개 데이터베이스. "작업 완료 위장(task completion pretense)" 패턴이 우리 케이스에 직접 대응.

### Context Window 관리 및 코드 생성 품질 (06-17 수정 관련)

6. Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2023). **Lost in the Middle: How Language Models Use Long Contexts.** arXiv:2307.03172.  
   — LLM이 긴 컨텍스트의 중간 부분을 사실상 무시하는 현상 실증. dep 파일이 예산 초과 시 드롭되는 것과 동일한 메커니즘. `_extract_api_surface()`로 핵심 정보 밀도를 높인 것의 이론적 근거.

7. Chen, M., Tworek, J., Jun, H., Yuan, Q., Pinto, H. P. de O., Kaplan, J., Edwards, H., Burda, Y., Joseph, N., Brockman, G., Ray, A., Puri, R., Krueger, G., Petrov, M., Khlaaf, H., Sastry, G., Mishkin, P., Chan, B., Gray, S., … Zaremba, W. (2021). **Evaluating Large Language Models Trained on Code.** arXiv:2107.03374.  
   — HumanEval 벤치마크 논문. LLM이 API 시그니처 없이 코드를 생성할 때 함수명/파라미터를 hallucinate하는 빈도를 보고함. Stage 3 deps 보강 및 `check_dataclass_fields()`의 필요성 근거.

### MLOps 성공 기준 표준 (06-23 수정 관련)

8. Zaharia, M., Chen, A., Davidson, A., Ghodsi, A., Hong, S. A., Konwinski, A., Murching, S., Nykodym, T., Ogilvie, P., Parkhe, M., Singh, A., & Xia, F. (2018). **Accelerating the Machine Learning Lifecycle with MLflow.** IEEE Data Engineering Bulletin, 41(4), 39–45.  
   — MLflow의 Run Status 모델(`RUNNING → FINISHED` 전환 조건: 실제 metric이 log됐을 때만)을 명시. **단, MLflow는 사용자가 사후 확인하는 tracking 시스템이고, 우리 L3는 런타임에서 분기하는 orchestration 로직이다.** "numeric metric 존재 여부를 성공 기준에 포함한다"는 철학만 참조.

9. Sculley, D., Holt, G., Golovin, D., Davydov, E., Phillips, T., Ebner, D., Chaudhary, V., Young, M., Crespo, J.-F., & Dennison, D. (2015). **Hidden Technical Debt in Machine Learning Systems.** NeurIPS 2015.  
   — ML 파이프라인에서 단일 성공 신호에 의존하는 시스템이 장기적으로 발생시키는 기술 부채를 분석. "Glue Code"와 "Pipeline Jungles" 문제가 success signal 오염과 연관됨.

### UI / 프론트엔드 (06-23 ApprovalDialog 수정 관련)

10. shadcn/ui. (2023). **Dialog — Radix UI 기반 접근 가능한 모달 다이얼로그 컴포넌트.** [https://ui.shadcn.com/docs/components/dialog](https://ui.shadcn.com/docs/components/dialog)  
    — `DialogContent`의 기본 className에 `sm:max-w-lg`가 하드코딩됨. 반응형 너비 override 시 `max-w-[arbitrary]` 임의값 클래스가 필요한 이유가 Tailwind CSS 충돌 해소 방식에 있음을 확인.
