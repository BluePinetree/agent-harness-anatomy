# Phase 1 설계 계획: CrewAI 네이티브 재설계

작성일: 2026-04-20  
상태: 설계 완료 (구현 진행 중)  
목적: `crewai_prototype`을 CrewAI 네이티브 구조(Agent + Task + Tool)로 재설계

---

## 1. 문제 정의

### 현재 구조의 한계

```
현재:
  ResearchCoordinator
    └── WorkerRunner
          └── PlannerWorker / CoderWorker / ... (자체 제작)
                └── CrewAI LLM (litellm.call() 만 사용)

문제:
  - CrewAI를 단순 LLM 래퍼로만 사용
  - 에이전트 간 Tool-Calling 없음
  - CoderWorker가 파일 생성 후 검증 루프를 직접 파이썬 코드로 처리
    → CrewAI가 아니라 워커 코드가 협업을 제어함
  - AutoGen / LangGraph와 비교 시 "프레임워크 비교"가 아닌 "래퍼 비교"가 됨
```

### 목표 구조

```
목표:
  ResearchCoordinatorV3 (반복 루프만 담당)
    └── ResearchCrew (CrewAI Crew)
          ├── PlannerAgent   → Task: 연구 계획 수립
          ├── DesignerAgent  → Task: 파일 구조 설계
          ├── CoderAgent     → Task: 파일 생성 (Tool 루프)
          │     tools: [Read, Write, List, SyntaxCheck, ImportCheck]
          ├── ExecutorAgent  → Task: 실험 실행
          │     tools: [RunCommand, ReadResult, Read]
          ├── AnalyzerAgent  → Task: 결과 분석
          │     tools: [Read, ReadResult]
          └── WriterAgent    → Task: 논문 작성
                tools: [Read, ReadResult, WriteReport]
```

---

## 2. 디렉토리 구조

```
crewai_prototype/
├── crew_tools/                  ← [신규] CrewAI BaseTool 구현체
│   ├── __init__.py
│   ├── workspace_tools.py       # Read / Write / List
│   ├── code_tools.py            # SyntaxCheck / ImportCheck
│   ├── execution_tools.py       # RunCommand / ReadResult
│   └── report_tools.py          # WriteReport
│
├── crew_agents/                 ← [신규] 에이전트 정의
│   ├── __init__.py
│   ├── planner.py
│   ├── designer.py
│   ├── coder.py
│   ├── executor.py
│   ├── analyzer.py
│   └── writer.py
│
├── crew/                        ← [신규] 크루 조립 + 코디네이터
│   ├── __init__.py
│   ├── callbacks.py             # step_callback / task_callback → UI 이벤트
│   ├── tasks.py                 # 6개 Task 정의
│   ├── research_crew.py         # Crew 조립
│   └── research_coordinator_v3.py  # 반복 루프 + 진입점
│
├── workers/                     ← [유지] legacy (폴백용)
├── orchestration/               ← [유지] V2 코디네이터
└── entrypoints/init.py          ← [수정] V3 코디네이터 추가
```

---

## 3. 컴포넌트 상세 설계

### 3-1. crew_tools — 도구 계층

모든 도구는 `crewai.tools.BaseTool`을 상속하며, Pydantic `args_schema`로 입력을 검증한다.

#### `workspace_tools.py`

| 도구 | 입력 | 출력 | 용도 |
|------|------|------|------|
| `WorkspaceReadTool` | `workspace_root`, `relative_path` | 파일 내용 (str) | Coder가 stable 파일 인터페이스 확인 |
| `WorkspaceWriteTool` | `workspace_root`, `relative_path`, `content` | "OK: wrote N lines" | Coder가 mutable 파일 작성 |
| `WorkspaceListTool` | `workspace_root`, `directory` | 파일 목록 (str) | Coder가 현재 상태 확인 |

**보안**: `_resolve()` 함수가 Path traversal(`../`) 차단. workspace root 밖 접근 시 에러 반환.

#### `code_tools.py`

| 도구 | 구현 | 출력 |
|------|------|------|
| `SyntaxCheckTool` | `py_compile.compile()` + tempfile | "OK" 또는 "SYNTAX_ERROR: ..." |
| `ImportCheckTool` | `subprocess python -c "import {module}"` (PYTHONPATH=src/) | "OK" 또는 "IMPORT_ERROR: ..." |

**ImportCheckTool 경로 처리**: `src/data.py` → module name `data`, `src/models/resnet.py` → `models.resnet`

#### `execution_tools.py`

| 도구 | 입력 | 출력 |
|------|------|------|
| `RunCommandTool` | `command`, `working_dir`, `timeout=300` | JSON `{return_code, duration_s, stdout, stderr}` |
| `ReadResultTool` | `result_path` | JSON 파일 내용 (str) |

**stdout/stderr 상한**: 4000자 잘라서 반환 (컨텍스트 오염 방지).

#### `report_tools.py`

| 도구 | 입력 | 출력 |
|------|------|------|
| `WriteReportTool` | `output_path`, `content`, `format="markdown"` | "OK: report saved to ..." |

---

### 3-2. crew_agents — 에이전트 계층

각 에이전트는 `make_<name>_agent(llm: LLM) -> Agent` 팩토리 함수로 생성.

#### PlannerAgent

```
role:      "AI Research Planner"
tools:     []
max_iter:  3
출력 형식: JSON (problem_statement, recommended_profile, primary_metric 등)
```

one-shot LLM 호출로 충분. 도구 없음.

#### DesignerAgent

```
role:      "Experiment Designer"
tools:     []
max_iter:  3
핵심 출력: workspace_structure.files + generation_order
```

`workspace_structure.files`에는 mutable 파일만 포함.  
stable 파일(`src/main.py`, `src/cli.py`, `src/artifacts.py`, `src/config_schema.py`)은 제외.

#### CoderAgent ← 핵심 변화

```
role:      "Research Code Engineer"
tools:     [WorkspaceReadTool, WorkspaceWriteTool, WorkspaceListTool,
            SyntaxCheckTool, ImportCheckTool]
max_iter:  30  ← 여러 파일 × 최대 3회 재시도
```

**Tool-Calling 루프 (에이전트가 자율적으로 수행)**:

```
for each file in generation_order:
  1. WorkspaceListTool → 현재 상태 확인
  2. WorkspaceReadTool → stable 파일 인터페이스 읽기
  3. WorkspaceWriteTool → 파일 작성
  4. SyntaxCheckTool → 문법 검증
     └─ 실패: 읽기 → 수정 → 재작성 (최대 3회)
  5. ImportCheckTool → import 검증
     └─ 실패: 읽기 → 수정 → 재작성 (최대 3회)
  6. 다음 파일로 이동
```

이 루프는 **에이전트의 LLM reasoning**으로 수행됨.  
외부 파이썬 코드(WorkerRunner 등)가 아님 → 이것이 V2와의 핵심 차이.

#### ExecutorAgent

```
role:      "Experiment Executor"
tools:     [RunCommandTool, ReadResultTool, WorkspaceReadTool]
max_iter:  5
```

실행 후 결과만 보고. 코드 수정은 하지 않음. 실패 시 stderr 전달.

#### AnalyzerAgent

```
role:      "Result Analyzer"
tools:     [WorkspaceReadTool, ReadResultTool]
max_iter:  5
출력 형식: JSON {execution_success, primary_metric_value,
                  failure_diagnosis, fix_instructions,
                  repair_actions, should_continue}
```

`should_continue=true` 조건: 실패가 fixable하고 `current_iteration < max_iterations`.

#### WriterAgent

```
role:      "Research Paper Writer"
tools:     [WorkspaceReadTool, ReadResultTool, WriteReportTool]
max_iter:  5
```

반복 루프 완료 후 **1회만** 실행. WriteReportTool로 `report.md` 저장.

---

### 3-3. crew — 조립 계층

#### `callbacks.py`

CrewAI의 `step_callback` / `task_callback`을 UI 이벤트 시스템에 연결.

```python
step_callback(agent_action) → emit("TOOL_CALL" or "AGENT_MESSAGE", ...)
task_callback(task_output)  → emit("PHASE_COMPLETE", ...)
```

`make_store_emitter(event_store, run_id, session_id)` → `(event_type, content, metadata) → None`  
EventStore에 RunEvent를 저장하는 클로저 반환.

#### `tasks.py`

6개 Task의 description 템플릿 + 팩토리 함수.

```
planning_task   → agent=planner,  context=[]
design_task     → agent=designer, context=[planning_task]
coding_task     → agent=coder,    context=[planning_task, design_task]
execution_task  → agent=executor, context=[coding_task]
analysis_task   → agent=analyzer, context=[execution_task]
writing_task    → agent=writer,   context=[planning_task, design_task,
                                           execution_task, analysis_task]
```

Task description에 `{workspace_root}`, `{generation_order}`, `{repair_context}` 등  
런타임 값을 주입하는 Jinja-like 슬롯 포함.

#### `research_crew.py`

```python
class ResearchCrew:
    def kickoff_main(inputs) → CrewOutput   # planner~analyzer (5 agents)
    def kickoff_writing(inputs) → CrewOutput # writer only (1 agent)
```

**Writer를 분리한 이유**: 반복 루프가 끝난 뒤 1회만 실행해야 함.  
main crew에 포함시키면 매 반복마다 논문을 쓰게 됨.

#### `research_coordinator_v3.py`

```python
class ResearchCoordinatorV3:
    def prepare_run(research_input) → V3PreparedRun   # 세션 생성
    def launch_prepared(prepared)   → None            # 백그라운드 스레드 실행
    def run_sync(research_input)    → dict            # CLI/테스트용 동기 실행

    def _execute(prepared):
        1. ScaffoldService로 stable 파일 생성
        2. make_store_emitter + make_step_callback + make_task_callback
        3. ResearchCrew 생성
        4. for iteration in range(1, max_iterations + 1):
               crew.kickoff_main(inputs)
               analysis = _extract_analysis(crew_output)
               if not analysis.should_continue: break
               repair_context = analysis.fix_instructions + repair_actions
        5. crew.kickoff_writing(inputs)
        6. session 상태 "completed" 업데이트
```

**반복 루프를 Coordinator가 관리하는 이유**:  
CrewAI의 `Process.sequential`은 조건부 반복을 지원하지 않음.  
`should_continue` 판단은 Python 레벨에서 처리.

---

## 4. 데이터 흐름

```
연구 입력 (JSON)
    │
    ▼
ResearchCoordinatorV3.prepare_run()
    │  → RunSession 생성 (run_id, session_id, status="queued")
    │  → SYSTEM_START 이벤트 발행
    ▼
ScaffoldService.materialize()
    │  → stable 파일 생성 (main.py, cli.py 등)
    │  → WORKSPACE_GENERATION_START 이벤트 발행
    ▼
[반복 루프 시작]
    │
    ├─ PlannerAgent → planning JSON
    │     ↓ context
    ├─ DesignerAgent → design JSON (workspace_structure 포함)
    │     ↓ context
    ├─ CoderAgent (tool loop)
    │     WorkspaceRead → WorkspaceWrite → SyntaxCheck → ImportCheck → ...
    │     ↓ context
    ├─ ExecutorAgent
    │     RunCommand → ReadResult
    │     ↓ context
    └─ AnalyzerAgent → {should_continue, fix_instructions}
         │
         ├─ should_continue=true → repair_context → 다음 반복
         └─ should_continue=false → 루프 종료
              │
              ▼
        WriterAgent (1회)
              WriteReport → report.md
              │
              ▼
        SYSTEM_END 이벤트 발행
```

---

## 5. 이벤트 발행 전략

### Coordinator 레벨 이벤트 (직접 발행)

| 이벤트 | 발행 시점 |
|--------|-----------|
| `SYSTEM_START` | prepare_run() |
| `WORKSPACE_GENERATION_START` | scaffold 완료 |
| `EXPERIMENT_START` | 각 반복 시작 |
| `EXPERIMENT_RESULT` | 각 반복 종료 |
| `SYSTEM_END` | 전체 완료 / 실패 |

### CrewAI Callback 이벤트 (자동 발행)

| 이벤트 | 트리거 |
|--------|--------|
| `TOOL_CALL` | `step_callback`: 도구 사용 시 |
| `AGENT_MESSAGE` | `step_callback`: 도구 없는 단계 |
| `PHASE_COMPLETE` | `task_callback`: 각 Task 완료 시 |

---

## 6. V2와의 공존 전략

### API 레벨 분리

```
POST /api/v1/research          → V2 (ResearchCoordinator, 기존 동작)
POST /api/v1/research?version=v3  → V3 (ResearchCoordinatorV3, CrewAI 네이티브)
```

### 초기화 조건

```python
# entrypoints/init.py
if OPENAI_API_KEY:
    coordinator_v3 = ResearchCoordinatorV3(session_store, event_store, ..., llm)
else:
    coordinator_v3 = None   # ?version=v3 요청 시 503 반환
```

### 기존 코드 처리

| 경로 | 처리 |
|------|------|
| `workers/` | 유지 (V2 폴백, legacy) |
| `orchestration/research_coordinator.py` | 유지 (V2 운영) |
| `orchestration/input_normalizer.py` | **V3에서도 재사용** |
| `workspace/scaffold_service.py` | **V3에서도 재사용** |
| `runtime/models.py`, `event_store.py` | **V3에서도 재사용** |
| `prompts/` | 참조용 유지 (Agent backstory 작성 시 참고) |
| `workers/coder/workspace_file_generator.py` | 장기적으로 CoderAgent로 대체 |

---

## 7. 미결 사항 및 결정 필요 항목

### 7-1. LLM 모델 선택 ✅ 결정됨

단일 LLM 방식으로 확정. 모델은 환경변수 `CREWAI_MODEL`로 지정 (기본값 `gpt-4o-mini`).  
에이전트별 다른 모델은 Phase 2 비교 실험 시 재검토.

### 7-2. CoderAgent max_iter

현재 `max_iter=30` (파일 5개 × 3회 재시도 × 도구 호출 2~3개).  
→ end-to-end 테스트 후 파일 수 기준으로 조정 예정.

### 7-3. generation_order 슬롯 주입 ✅ 결정됨

`inputs["generation_order"] = ""`로 유지 (빈 문자열).  
CoderAgent는 `context=[planning_task, design_task]`를 통해 DesignerAgent의 실제 출력을 직접 읽음.  
Task description의 `{generation_order}` 슬롯은 보조 힌트 — 비어있어도 context가 우선.

### 7-4. Writer 분리 trade-off ✅ 결정됨

분리 방식 유지. WriterAgent가 `WorkspaceReadTool` / `ReadResultTool`로 필요한 정보를 직접 수집.  
inputs에 `workspace_root`, `report_path`, `research_topic`, `research_goal` 포함되어 있어 충분한 시작점 제공.

### 7-5. 테스트 전략

Phase 1 완료 기준:

```
□ crew_tools 단위 테스트
  - WorkspaceWriteTool → 파일 실제 생성 확인
  - SyntaxCheckTool → 잘못된 코드 SYNTAX_ERROR 반환 확인
  - ImportCheckTool → 정상/비정상 모듈 구분 확인
  - RunCommandTool → return_code 0/1 반환 확인

□ end-to-end 테스트 (사용자 직접 수행)
  - CIFAR-100 또는 generic_script 프로파일로 실행
  - report.md 생성 확인
  - UI 이벤트 스트림 확인
```

---

## 8. 구현 순서 (파일별)

| # | 파일 | 상태 |
|---|------|------|
| 1 | `crew_tools/__init__.py` | ✅ 완료 |
| 2 | `crew_tools/workspace_tools.py` | ✅ 완료 |
| 3 | `crew_tools/code_tools.py` | ✅ 완료 |
| 4 | `crew_tools/execution_tools.py` | ✅ 완료 |
| 5 | `crew_tools/report_tools.py` | ✅ 완료 |
| 6 | `crew_agents/__init__.py` | ✅ 완료 |
| 7 | `crew_agents/planner.py` | ✅ 완료 |
| 8 | `crew_agents/designer.py` | ✅ 완료 |
| 9 | `crew_agents/coder.py` | ✅ 완료 |
| 10 | `crew_agents/executor.py` | ✅ 완료 |
| 11 | `crew_agents/analyzer.py` | ✅ 완료 |
| 12 | `crew_agents/writer.py` | ✅ 완료 |
| 13 | `crew/callbacks.py` | ✅ 완료 (metadata mutation 버그 수정) |
| 14 | `crew/tasks.py` | ✅ 완료 |
| 15 | `crew/research_crew.py` | ✅ 완료 (매 kickoff마다 fresh agent 생성) |
| 16 | `crew/research_coordinator_v3.py` | ✅ 완료 (constraints 슬롯 보장) |
| 17 | `entrypoints/init.py` | ✅ 완료 |
| 18 | `api/routes/research.py` | ✅ 완료 |
| 19 | end-to-end 테스트 (CIFAR-100) | 사용자 직접 수행 예정 |

---

## 9. 다음 단계

Phase 1이 안정화되면 `COMPARISON_PLAN.md`의 Phase 2로 진행:

1. **공통 기준 수립**: 입력 스펙, 출력 스펙, 8개 평가 지표 확정
2. **AutoGen 구현**: GroupChat + 동일한 도구 로직 (`register_function` 방식)
3. **LangGraph 구현**: StateGraph + ReAct Coder 노드
4. **비교 실험**: 동일 태스크로 3개 프레임워크 실행 후 지표 비교
