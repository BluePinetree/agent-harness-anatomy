# 멀티 에이전트 AI 연구 시스템 비교 계획 (Option C)
# CrewAI 네이티브 안정화 → AutoGen → LangGraph → 비교 분석

작성일: 2026-04-20  
목표: 동일한 연구 작업을 3개 프레임워크로 수행하고, 에이전트 협업 방식·결과물·효율을 비교

---

## 0. 핵심 전제

### 현재 문제
```
crewai_prototype의 현재 구조:
  ResearchCoordinator (자체 제작 오케스트레이터)
    → WorkerRunner (자체 제작)
      → PlannerWorker, CoderWorker ... (자체 제작)
        → CrewAI LLM (단순 llm.call() 만 사용)

문제: CrewAI를 LLM 래퍼로만 사용 → 프레임워크 비교 의미 없음
     Agent 간 tool-calling 없음 → 파일 생성/검증 루프 불가
```

### 목표 구조 (Option C — 하이브리드)
```
Phase 1: crewai_prototype을 CrewAI 네이티브로 재설계
  - Agent + Task + Tool 구조로 전환
  - 각 에이전트가 직접 도구를 호출 (파일 읽기/쓰기/실행/검증)
  - 안정적인 end-to-end 연구 파이프라인 완성

Phase 2: 공통 기준 수립 (입력 스펙, 출력 스펙, 평가 지표)

Phase 3: AutoGen 구현 (같은 작업, 다른 협업 방식)

Phase 4: LangGraph 구현 (같은 작업, 그래프 기반)

Phase 5: 비교 분석
```

---

## Phase 1: CrewAI 네이티브 재설계 + 안정화

### 1-1. 아키텍처 변경 전/후

```
[현재]
ResearchCoordinator
  ├── PlannerWorker.run(payload)  → one-shot LLM call
  ├── DesignerWorker.run(payload) → one-shot LLM call
  ├── CoderWorker.run(payload)    → LLM + workspace_file_generator (새로 구현)
  ├── ExecutorWorker.run(payload) → subprocess 실행
  ├── AnalyzerWorker.run(payload) → one-shot LLM call
  └── WriterWorker.run(payload)   → one-shot LLM call

[목표]
ResearchCrew (CrewAI Crew)
  ├── PlannerAgent   tools=[]                    → Task: 연구 계획 수립
  ├── DesignerAgent  tools=[]                    → Task: 실험 설계 + 파일 구조 설계
  ├── CoderAgent     tools=[Read, Write, Syntax, Import, ListDir]
  │                                               → Task: 파일 생성 (tool loop)
  ├── ExecutorAgent  tools=[RunCommand]           → Task: 실험 실행
  ├── AnalyzerAgent  tools=[Read]                 → Task: 결과 분석
  └── WriterAgent    tools=[Read, WriteReport]    → Task: 논문 작성
```

### 1-2. 도구(Tool) 구현

파일: `crewai_prototype/crew_tools/`

#### `workspace_tools.py`
```python
# WorkspaceReadTool
# - 입력: relative_path (e.g. "src/data.py")
# - 출력: 파일 내용 (str)
# - 용도: Coder가 stable 파일 인터페이스 확인, Analyzer가 결과 파일 읽기

# WorkspaceWriteTool  
# - 입력: relative_path, content
# - 출력: 성공/실패 메시지
# - 용도: Coder가 mutable 파일 작성

# WorkspaceListTool
# - 입력: directory (e.g. "src/")
# - 출력: 파일 목록
# - 용도: Coder가 현재 workspace 상태 확인
```

#### `code_tools.py`
```python
# SyntaxCheckTool
# - 입력: relative_path
# - 출력: "OK" 또는 에러 메시지
# - 구현: py_compile.compile()

# ImportCheckTool
# - 입력: relative_path
# - 출력: "OK" 또는 에러 메시지
# - 구현: subprocess python -c "import {module}"
```

#### `execution_tools.py`
```python
# RunCommandTool
# - 입력: command (str), working_dir (str), timeout (int)
# - 출력: stdout, stderr, return_code, duration
# - 용도: ExecutorAgent가 실험 실행

# ReadResultTool
# - 입력: result_path (e.g. "results/result.json")
# - 출력: JSON 내용
# - 용도: ExecutorAgent/AnalyzerAgent가 결과 확인
```

#### `report_tools.py`
```python
# WriteReportTool
# - 입력: content (str), format ("markdown" | "pdf")
# - 출력: 저장 경로
# - 용도: WriterAgent가 최종 보고서 작성
```

### 1-3. 에이전트(Agent) 구현

파일: `crewai_prototype/crew_agents/`

#### `planner.py` — PlannerAgent
```
role: "AI 연구 기획자"
goal: "연구 목표를 구체적인 실험 계획으로 변환"
backstory: "AI 연구 분야의 10년 경력 기획자. 논문 목표 기준으로
            실험 전략과 평가 방법을 설계한다."
tools: []  # 계획은 one-shot으로 충분
output_format: PlanningOutput (JSON schema)
  - recommended_profile
  - experiment_strategy  
  - primary_metric
  - scope_constraints
```

#### `designer.py` — DesignerAgent
```
role: "실험 설계자"
goal: "연구 계획을 실행 가능한 실험 구조와 코드 파일 설계로 변환"
backstory: "ML 실험 시스템 설계 전문가. 어떤 파일이 필요하고
            어떤 순서로 작성해야 하는지 정확히 안다."
tools: []  # 설계는 one-shot
output_format: DesignOutput (JSON schema)
  - experiment_family
  - evaluation_protocol
  - workspace_structure  ← 핵심: 어떤 파일을 어떤 순서로 만들지
    - files: [{path, responsibility, symbols, dependencies}]
    - generation_order: [path, ...]
```

#### `coder.py` — CoderAgent  ← 핵심 변화
```
role: "연구 코드 엔지니어"
goal: "설계된 파일 구조대로 동작하는 Python 코드를 작성하고
       각 파일이 syntax/import 검증을 통과할 때까지 수정한다"
backstory: "Python 전문 개발자. 파일을 하나씩 작성하고 즉시
            검증하며, 오류가 있으면 그 자리에서 수정한다."
tools: [WorkspaceReadTool, WorkspaceWriteTool, WorkspaceListTool,
        SyntaxCheckTool, ImportCheckTool]

동작 방식 (tool-calling loop):
  1. WorkspaceListTool로 현재 파일 확인
  2. WorkspaceReadTool로 stable 파일 인터페이스 읽기
  3. 첫 번째 mutable 파일 작성 (WorkspaceWriteTool)
  4. SyntaxCheckTool로 검증
  5. 실패 시 수정 후 재검증 (최대 3회)
  6. 성공 시 ImportCheckTool로 import 검증
  7. 다음 파일로 이동 (generation_order 순서대로)
  8. 모든 파일 완료 후 완료 보고
```

#### `executor.py` — ExecutorAgent
```
role: "실험 실행자"
goal: "작성된 코드를 실행하고 결과를 수집한다"
backstory: "실험 환경 관리 전문가. 코드를 실행하고 결과가
            올바른 형식인지 확인한다."
tools: [RunCommandTool, ReadResultTool, WorkspaceReadTool]

동작 방식:
  1. RunCommandTool로 main.py 실행
  2. ReadResultTool로 results/result.json 확인
  3. 실패 시 stderr 분석 후 실행 결과 보고 (수정은 Coder에게)
```

#### `analyzer.py` — AnalyzerAgent
```
role: "결과 분석가"
goal: "실험 결과를 분석하고 개선 방향을 제시한다"
backstory: "데이터 분석 전문가. 수치 결과를 해석하고
            다음 반복에서 개선할 점을 명확히 제시한다."
tools: [WorkspaceReadTool, ReadResultTool]

output_format: AnalysisOutput
  - execution_success: bool
  - primary_metric_value: float
  - failure_diagnosis: str
  - fix_instructions: [str]
  - repair_actions: [{path, symbol, reason}]
  - should_continue: bool  ← 다음 iteration 여부 결정
```

#### `writer.py` — WriterAgent
```
role: "연구 논문 작성자"
goal: "연구 과정과 결과를 학술 논문 형식으로 작성한다"
backstory: "ML 논문 전문 작성자. 실험 목표, 방법, 결과,
            결론을 명확하게 서술하며 재현 가능성을 강조한다."
tools: [WorkspaceReadTool, ReadResultTool, WriteReportTool]

output_format: 논문 (Markdown)
  - Abstract
  - Introduction
  - Methodology
  - Experimental Setup
  - Results & Discussion
  - Conclusion
  - Appendix (코드 구조, 실행 환경)
```

### 1-4. Task + Crew 구성

파일: `crewai_prototype/crew/research_crew.py`

```python
# Task 정의 (순서대로)
planning_task = Task(
    description="""
    다음 연구 주제를 분석하고 실험 계획을 수립하라:
    연구 주제: {research_topic}
    연구 목표: {research_goal}
    도메인: {research_domain}
    
    출력: JSON 형식의 실험 계획
    """,
    expected_output="JSON 형식 계획서 (recommended_profile, primary_metric 포함)",
    agent=planner_agent
)

design_task = Task(
    description="""
    연구 계획을 바탕으로 실험을 설계하라.
    Context: {planning_output}
    
    반드시 workspace_structure를 포함할 것:
    - 필요한 파일 목록 (stable 파일 제외)
    - 각 파일의 책임, 주요 함수, 의존성
    - 생성 순서 (의존성 없는 파일 먼저)
    """,
    expected_output="JSON 형식 설계서 (workspace_structure 포함)",
    agent=designer_agent,
    context=[planning_task]
)

coding_task = Task(
    description="""
    설계된 파일 구조대로 workspace에 코드를 작성하라.
    Workspace 경로: {workspace_root}
    파일 생성 순서: {generation_order}
    
    각 파일 작성 후 반드시:
    1. SyntaxCheckTool로 문법 검증
    2. ImportCheckTool로 import 검증
    3. 실패 시 즉시 수정 (최대 3회)
    """,
    expected_output="생성된 파일 목록과 각 파일의 검증 결과",
    agent=coder_agent,
    context=[planning_task, design_task]
)

execution_task = Task(
    description="""
    작성된 코드를 실행하고 결과를 수집하라.
    실행 명령: {run_command}
    결과 경로: {workspace_root}/results/result.json
    """,
    expected_output="실행 결과 요약 (성공/실패, 주요 지표, 에러 로그)",
    agent=executor_agent,
    context=[coding_task]
)

analysis_task = Task(
    description="""
    실험 결과를 분석하고 평가하라.
    추가 반복이 필요하다면 구체적인 수정 지침을 제시하라.
    최대 반복 횟수: {max_iterations}
    현재 반복: {current_iteration}
    """,
    expected_output="분석 결과 (should_continue, fix_instructions, repair_actions)",
    agent=analyzer_agent,
    context=[execution_task]
)

writing_task = Task(
    description="""
    전체 연구 과정과 결과를 학술 논문 형식으로 작성하라.
    보고서 저장 경로: {output_root}/report.md
    """,
    expected_output="완성된 연구 보고서 (Markdown 형식)",
    agent=writer_agent,
    context=[planning_task, design_task, execution_task, analysis_task]
)

# Crew 구성
research_crew = Crew(
    agents=[planner, designer, coder, executor, analyzer, writer],
    tasks=[planning_task, design_task, coding_task, execution_task, analysis_task, writing_task],
    process=Process.sequential,
    step_callback=emit_step_event,   # UI 이벤트 연결
    task_callback=emit_task_event,
    verbose=True
)
```

### 1-5. 반복(Iteration) 루프 처리

CrewAI 자체는 조건부 반복을 지원하지 않으므로, **Coordinator 레벨에서 루프**를 관리:

```python
# crewai_prototype/crew/research_coordinator_v3.py

class ResearchCoordinatorV3:
    def run(self, research_input: dict) -> ResearchResult:
        max_iterations = research_input.get("max_iterations", 3)
        
        for iteration in range(1, max_iterations + 1):
            # 1. Planner + Designer + Coder + Executor + Analyzer 실행
            result = self.crew.kickoff(inputs={
                **research_input,
                "current_iteration": iteration,
                "workspace_root": self.workspace_root,
            })
            
            analysis = result.tasks_output[-1]  # AnalyzerAgent 결과
            
            # 2. 분석 결과 확인
            if not analysis.should_continue or iteration == max_iterations:
                break
            
            # 3. 다음 반복: repair_context를 CoderTask에 주입
            research_input["repair_context"] = {
                "fix_instructions": analysis.fix_instructions,
                "repair_actions": analysis.repair_actions,
                "source_iteration": iteration,
            }
        
        # 4. Writer 실행 (반복 완료 후 1회)
        report = self.writer_crew.kickoff(inputs={...})
        return ResearchResult(report=report, iterations=iteration)
```

### 1-6. UI 이벤트 연결

```python
# crewai_prototype/crew/callbacks.py

def make_step_callback(event_service, run_id, session_id):
    """CrewAI step_callback → UI 이벤트 발행"""
    def on_step(agent_action):
        event_service.emit(
            run_id=run_id,
            session_id=session_id,
            event_type="AGENT_MESSAGE",
            content=f"{agent_action.tool}: {agent_action.tool_input}",
            metadata={
                "agent": agent_action.agent,
                "tool": agent_action.tool,
                "tool_input": agent_action.tool_input,
            }
        )
    return on_step

def make_task_callback(event_service, run_id, session_id):
    """CrewAI task_callback → UI PHASE_COMPLETE 이벤트"""
    def on_task(task_output):
        event_service.emit(
            run_id=run_id,
            session_id=session_id,
            event_type="PHASE_COMPLETE",
            content=task_output.description[:100],
            metadata={"raw_output": task_output.raw}
        )
    return on_task
```

### 1-7. 구현 순서 (파일별)

| 순서 | 파일 | 작업 | 비고 |
|------|------|------|------|
| 1 | `crew_tools/__init__.py` | 패키지 생성 | |
| 2 | `crew_tools/workspace_tools.py` | Read/Write/List 도구 | CrewAI BaseTool 상속 |
| 3 | `crew_tools/code_tools.py` | SyntaxCheck/ImportCheck | py_compile, subprocess |
| 4 | `crew_tools/execution_tools.py` | RunCommand/ReadResult | |
| 5 | `crew_tools/report_tools.py` | WriteReport | |
| 6 | `crew_agents/__init__.py` | 패키지 생성 | |
| 7 | `crew_agents/planner.py` | PlannerAgent 정의 | |
| 8 | `crew_agents/designer.py` | DesignerAgent 정의 | workspace_structure 출력 |
| 9 | `crew_agents/coder.py` | CoderAgent 정의 | tool-calling 핵심 |
| 10 | `crew_agents/executor.py` | ExecutorAgent 정의 | |
| 11 | `crew_agents/analyzer.py` | AnalyzerAgent 정의 | should_continue 판단 |
| 12 | `crew_agents/writer.py` | WriterAgent 정의 | |
| 13 | `crew/callbacks.py` | step/task callback | UI 이벤트 연결 |
| 14 | `crew/tasks.py` | Task 정의 6개 | context 연결 포함 |
| 15 | `crew/research_crew.py` | Crew 조립 | Process.sequential |
| 16 | `crew/research_coordinator_v3.py` | 반복 루프 + 진입점 | |
| 17 | `entrypoints/init.py` 수정 | v3 coordinator 연결 | |
| 18 | end-to-end 테스트 | CIFAR-100 작업으로 검증 | |

### 1-8. 기존 코드 처리

```
유지 (참조/폴백용):
  workers/          → 삭제하지 않고 legacy/ 로 이동
  orchestration/research_coordinator.py → legacy로 이동

유지 (계속 사용):
  orchestration/input_normalizer.py
  runtime/models.py
  runtime/event_service.py
  workspace/scaffold_service.py (stable 파일 생성에 여전히 사용)
  profiles/
  core/
  prompts/ (참조용 — Agent backstory/task description에 활용)

삭제 검토:
  workers/coder/workspace_file_generator.py → CoderAgent tool loop로 대체
  workers/coder/types.py                    → 불필요
  prompts/coder.py의 build_file_generation_prompt 등 → CoderAgent가 직접 처리
```

---

## Phase 2: 공통 비교 기준 수립

### 2-1. 공통 연구 입력 스펙

```json
{
  "research_topic": "CIFAR-100 이미지 분류 실험",
  "research_goal": "ResNet-18 기반 모델로 Top-1 Accuracy 60% 이상 달성",
  "research_domain": "computer vision / image classification",
  "primary_metric": "top1_accuracy",
  "max_iterations": 3,
  "data_config": {
    "dataset": "cifar100",
    "data_root": "../../.cache/cifar100",
    "download": true
  },
  "compute_config": {
    "device": "cpu",
    "epochs": 1,
    "batch_size": 32,
    "num_workers": 0,
    "seed": 42
  }
}
```

### 2-2. 공통 출력 스펙 (논문 포맷)

```markdown
# {연구 제목}

## Abstract (200단어 이내)

## 1. Introduction
- 연구 배경
- 문제 정의
- 기여점

## 2. Methodology
- 실험 설계
- 모델 구조
- 평가 방법

## 3. Experimental Setup
- 데이터셋
- 하이퍼파라미터
- 실행 환경

## 4. Results & Discussion
- 주요 지표 (표 포함)
- 반복별 개선 과정
- 실패 분석 (있는 경우)

## 5. Conclusion

## Appendix
- 코드 구조
- 실행 명령
- 에이전트 실행 로그 요약
```

### 2-3. 평가 지표

| 지표 | 측정 방법 | 비고 |
|------|-----------|------|
| **완료율** | 오류 없이 end-to-end 완료 비율 | 3회 실행 평균 |
| **1차 성공률** | iteration 1에서 실행 성공 비율 | 코드 품질 지표 |
| **평균 반복 횟수** | 목표 달성까지 필요한 iteration 수 | |
| **실행 시간** | 총 소요 시간 (초) | |
| **토큰 사용량** | 총 input + output tokens | 비용 지표 |
| **논문 품질** | LLM-judge 점수 (1-5) | GPT-4 평가 |
| **코드 테스트 통과율** | contract smoke test 통과 비율 | |
| **에이전트 협력 패턴** | 정성적 분석 | 대화 로그 분석 |

### 2-4. 벤치마크 작업 목록

| 작업 | 도메인 | 난이도 | 핵심 평가 |
|------|--------|--------|-----------|
| **Task 1**: CIFAR-100 분류 | Vision | 중 | 코드 생성 + 학습 실행 |
| **Task 2**: Titanic 생존 예측 | Tabular | 하 | 전처리 + 분류기 |
| **Task 3**: 주가 예측 (단변량) | TimeSeries | 중 | 시계열 처리 |
| **Task 4**: 텍스트 감성 분류 | NLP | 중 | 자연어 처리 |
| **Task 5**: 임의 Python 스크립트 검증 | Generic | 하 | 범용성 테스트 |

---

## Phase 3: AutoGen 구현

### 3-1. AutoGen의 접근 방식

```
핵심 특징:
- 에이전트 간 자유로운 대화 (GroupChat)
- UserProxyAgent가 코드를 자동 실행
- 에러 발생 시 자동으로 관련 에이전트에게 수정 요청
- 사람(human_input_mode)을 루프에 포함 가능
```

### 3-2. 에이전트 구성

```python
# autogen_prototype/research_team.py

# 1. 연구 팀장 (orchestration)
research_manager = AssistantAgent(
    name="ResearchManager",
    system_message="연구 팀을 이끄는 팀장. 작업을 분배하고 진행을 조율한다.",
    llm_config=llm_config
)

# 2. 계획/설계 (합쳐서 1명)
research_planner = AssistantAgent(
    name="ResearchPlanner",
    system_message="연구 계획 및 실험 설계 전문가.",
    llm_config=llm_config
)

# 3. 코더 (tool use 포함)
research_coder = AssistantAgent(
    name="ResearchCoder",
    system_message="Python 코드 작성 전문가. 파일을 작성하고 검증한다.",
    llm_config={**llm_config, "tools": code_tools}
)

# 4. 실행자 (코드 자동 실행)
code_executor = UserProxyAgent(
    name="CodeExecutor",
    human_input_mode="NEVER",
    code_execution_config={"work_dir": workspace_root, "use_docker": False}
)

# 5. 분석가 + 작성자
research_analyst = AssistantAgent(
    name="ResearchAnalyst",
    system_message="결과 분석 및 논문 작성 전문가.",
    llm_config=llm_config
)

# GroupChat 구성
group_chat = GroupChat(
    agents=[research_manager, research_planner, research_coder, 
            code_executor, research_analyst],
    messages=[],
    max_round=30,
    speaker_selection_method="auto"  # LLM이 다음 발언자 결정
)
```

### 3-3. 파일 구조

```
autogen_prototype/
├── agents/
│   ├── manager.py
│   ├── planner.py
│   ├── coder.py
│   ├── executor.py       # UserProxyAgent
│   └── analyst.py
├── tools/
│   ├── code_tools.py     # CrewAI와 동일한 도구, AutoGen 형식으로
│   └── file_tools.py
├── research_team.py      # GroupChat 구성
├── coordinator.py        # 진입점 + UI 이벤트 연결
└── config.py
```

### 3-4. CrewAI와의 핵심 차이점

| 항목 | CrewAI | AutoGen |
|------|--------|---------|
| 협업 방식 | 순차적 Task 실행 | 자유 대화 (GroupChat) |
| 코드 실행 | ExecutorAgent가 Tool 호출 | UserProxyAgent 자동 실행 |
| 오류 수정 | Coder가 Tool로 직접 수정 | 대화를 통해 Coder에게 요청 |
| 반복 루프 | 외부 Coordinator가 관리 | GroupChat 내에서 자연스럽게 |
| 제어 가능성 | 높음 (명시적 순서) | 낮음 (LLM이 흐름 결정) |

---

## Phase 4: LangGraph 구현

### 4-1. LangGraph의 접근 방식

```
핵심 특징:
- 명시적 상태 그래프 (StateGraph)
- 조건부 엣지로 분기/루프 정의
- 체크포인팅으로 중단/재시작 가능
- 각 노드가 독립적인 함수 (테스트 용이)
```

### 4-2. 상태 정의

```python
# langgraph_prototype/state.py

class ResearchState(TypedDict):
    # 입력
    research_input: dict
    
    # 각 단계 출력
    planning_output: dict
    design_output: dict
    coding_output: dict
    execution_output: dict
    analysis_output: dict
    report: str
    
    # 제어
    iteration: int
    max_iterations: int
    should_continue: bool
    error_context: dict
    
    # 메타
    run_id: str
    messages: list  # 에이전트 간 메시지 (선택)
```

### 4-3. 그래프 구조

```python
# langgraph_prototype/research_graph.py

graph = StateGraph(ResearchState)

# 노드 추가
graph.add_node("planner",  planner_node)   # LLM + 출력 파싱
graph.add_node("designer", designer_node)
graph.add_node("coder",    coder_node)      # LLM + tool 반복
graph.add_node("executor", executor_node)   # subprocess 실행
graph.add_node("analyzer", analyzer_node)
graph.add_node("writer",   writer_node)

# 엣지 (기본 순서)
graph.set_entry_point("planner")
graph.add_edge("planner",  "designer")
graph.add_edge("designer", "coder")
graph.add_edge("coder",    "executor")
graph.add_edge("executor", "analyzer")

# 조건부 엣지 (analyzer 이후 분기)
graph.add_conditional_edges(
    "analyzer",
    should_continue_research,  # 판단 함수
    {
        "continue": "coder",   # 다음 iteration
        "done":     "writer",  # 완료 → 논문 작성
    }
)
graph.add_edge("writer", END)

# 컴파일 (체크포인팅 포함)
app = graph.compile(checkpointer=MemorySaver())
```

### 4-4. Coder 노드 (tool-calling loop)

```python
# langgraph_prototype/nodes/coder.py

def coder_node(state: ResearchState) -> ResearchState:
    """LLM + tool 반복으로 파일 생성"""
    workspace_structure = state["design_output"]["workspace_structure"]
    repair_context = state.get("error_context", {})
    
    # LangGraph의 ToolNode + ReAct 패턴 활용
    agent = create_react_agent(llm, tools=[
        read_workspace_file,
        write_workspace_file,
        check_syntax,
        check_import,
        list_workspace_files,
    ])
    
    result = agent.invoke({
        "messages": [HumanMessage(content=build_coder_prompt(state))]
    })
    
    return {**state, "coding_output": parse_coder_result(result)}
```

### 4-5. 파일 구조

```
langgraph_prototype/
├── state.py              # ResearchState TypedDict
├── research_graph.py     # 그래프 정의 및 컴파일
├── nodes/
│   ├── planner.py
│   ├── designer.py
│   ├── coder.py          # ReAct 에이전트 패턴
│   ├── executor.py
│   ├── analyzer.py
│   └── writer.py
├── tools/
│   ├── workspace_tools.py  # CrewAI/AutoGen과 동일한 도구
│   └── execution_tools.py
├── edges/
│   └── conditions.py     # 조건부 엣지 판단 함수
├── coordinator.py        # 진입점 + UI 이벤트 연결
└── config.py
```

### 4-6. CrewAI/AutoGen과의 핵심 차이점

| 항목 | CrewAI | AutoGen | LangGraph |
|------|--------|---------|-----------|
| 흐름 제어 | Task 순서 정의 | LLM이 결정 | 명시적 그래프 |
| 분기/루프 | 외부 Coordinator | GroupChat 내 | add_conditional_edges |
| 체크포인팅 | 없음 | 없음 | 내장 (MemorySaver) |
| 디버깅 | verbose 로그 | 대화 로그 | 그래프 시각화 가능 |
| 확장성 | Agent/Tool 추가 | Agent 추가 | 노드/엣지 추가 |
| 재시작 | 처음부터 | 처음부터 | 중단 지점부터 |

---

## Phase 5: 비교 분석

### 5-1. 자동화 비교 실험

```python
# comparison/run_comparison.py

for task in benchmark_tasks:
    for framework in ["crewai", "autogen", "langgraph"]:
        for trial in range(3):
            result = run_research(framework, task, trial)
            save_result(framework, task, trial, result)

# 결과 집계
report = generate_comparison_report(all_results)
```

### 5-2. 측정 항목별 집계

```
정량적 지표 (자동 측정):
├── 완료율 (%)
├── 1차 성공률 (%)
├── 평균 반복 횟수
├── 평균 실행 시간 (초)
├── 평균 토큰 사용량
├── 평균 API 비용 (USD)
└── 코드 테스트 통과율 (%)

정성적 지표 (LLM-judge):
├── 논문 완성도 (1-5)
├── 논문 정확성 (1-5)
├── 에이전트 협력 자연스러움 (1-5)
└── 결과 재현 가능성 (1-5)
```

### 5-3. 비교 분석 문서 구조

```
comparison/
├── results/
│   ├── crewai/   {task}_{trial}.json
│   ├── autogen/  {task}_{trial}.json
│   └── langgraph/{task}_{trial}.json
├── analysis/
│   ├── quantitative_comparison.md
│   ├── qualitative_comparison.md
│   └── agent_behavior_analysis.md
└── final_report.md
```

---

## 전체 타임라인

| Phase | 내용 | 예상 소요 | 완료 기준 |
|-------|------|-----------|-----------|
| **Phase 1** | CrewAI 네이티브 재설계 | 3-4주 | CIFAR-100 end-to-end 3회 성공 |
| **Phase 2** | 공통 기준 확정 | 1주 | 5개 벤치마크 작업 정의 완료 |
| **Phase 3** | AutoGen 구현 | 2-3주 | Phase 1과 동일 작업 실행 가능 |
| **Phase 4** | LangGraph 구현 | 2-3주 | Phase 1과 동일 작업 실행 가능 |
| **Phase 5** | 비교 분석 | 1-2주 | 최종 비교 리포트 완성 |

---

## 당장 시작할 것 (Phase 1 Step 1)

```
1. crewai_prototype/crew_tools/ 디렉토리 생성
2. WorkspaceReadTool, WorkspaceWriteTool, WorkspaceListTool 구현
   (CrewAI BaseTool 상속, 기존 workspace_file_generator.py 로직 활용)
3. SyntaxCheckTool, ImportCheckTool 구현
4. 도구 단위 테스트 작성 + 통과 확인
5. CoderAgent 정의 (도구 연결)
6. CoderAgent 단독 테스트 (단일 파일 생성 → 검증)
```

---

## 참고: 프레임워크별 Tool 구현 형식

### CrewAI
```python
from crewai.tools import BaseTool

class WorkspaceReadTool(BaseTool):
    name: str = "workspace_read"
    description: str = "workspace 내 파일을 읽는다"
    
    def _run(self, relative_path: str) -> str:
        ...
```

### AutoGen
```python
from autogen import register_function

def workspace_read(relative_path: str) -> str:
    """workspace 내 파일을 읽는다"""
    ...

register_function(workspace_read, caller=coder_agent, executor=code_executor)
```

### LangGraph
```python
from langchain_core.tools import tool

@tool
def workspace_read(relative_path: str) -> str:
    """workspace 내 파일을 읽는다"""
    ...

# create_react_agent(llm, tools=[workspace_read, ...])
```

같은 핵심 로직(파일 읽기/쓰기/검증)이 각 프레임워크 형식으로 래핑됨.
이것이 비교 실험의 공정성을 보장하는 핵심.
