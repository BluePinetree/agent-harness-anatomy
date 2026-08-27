# Research System 구현 계획

> **출처 정리 안내 (2026-08-20)**
>
> 이 문서가 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했습니다(커밋 이력에는 없었음). 그에 따라 **유출 트리를 가리키던
> 파일 경로와 줄번호 인용을 정리**했습니다. 분석 내용 자체는 본인이 쓴 것으로 그대로 둡니다.
>
> 공식 문서: <https://docs.claude.com/en/docs/claude-code>

---

> 작성일: 2026-04-24  
> 목표: CrewAI 프로토타입 안정화 + LangGraph 프로토타입 완성 (Claude Code 아키텍처 차용)

---

## Part 1 — CrewAI 프로토타입 최소 안정화

### 근본 원인 요약

CrewAI가 불안정한 이유는 **도구 호출이 텍스트 파싱에 의존**하기 때문이다.  
CrewAI `parser.py:181`이 배열 입력을 수리 없이 통과시키고, Pydantic이 `list`를 `dict`로 받아 validation error를 낸다. LLM은 이를 "결과 없음"으로 해석해 날조한다.

```
LLM 출력: Action Input: [{"path": "src/data.py"}, ...]
                    ↓
parser.py:181  → 배열 → json_repair 스킵 → Pydantic에 list 전달
                    ↓
ValidationError → LLM이 결과 날조 → 파이프라인 오염
```

### 수정 완료 목록 (이번 세션)

| # | 파일 | 수정 내용 |
|---|------|---------|
| ✅ 1 | `crew_tools/workspace_tools.py` | `_ArraySafeModel`: `model_validator(mode='before')`로 배열 입력 시 첫 번째 요소 추출 |
| ✅ 2 | `crew_tools/code_tools.py` | 동일한 `_ArraySafeModel` 적용 |
| ✅ 3 | `crew_tools/execution_tools.py` | 동일한 `_ArraySafeModel` 적용 |
| ✅ 4 | `crew/tasks.py` | `_EXECUTION_DESC`: RunCommandTool 관측값에 `return_code` 없으면 return_code=-2 보고, 결과 날조 금지 |
| ✅ 5 | `crew/tasks.py` | `_CODING_DESC` / `_CODING_REPAIR_DESC`: "ONE tool call per turn, never array" 규칙 추가 |
| ✅ 6 | `crew/tasks.py` | `_CODING_REPAIR_DESC`: Final Answer로 코드 내용 출력 금지 |
| ✅ 7 | `crew/research_coordinator_v3.py` | `_build_coder_designer_context()`: domain 배너 + "sklearn 금지, flat paths" 명시 |
| ✅ 8 | `crew/research_coordinator_v3.py` | `_load_manifest_from_disk()`: `bundle=None`이어도 disk에서 전체 mutable files 체크 |
| ✅ 9 | `crew/research_coordinator_v3.py` | `_format_file_list()`: mutable_files를 numbered list로 전달 (배열 batching 억제) |
| ✅ 10 | `crew_agents/coder.py` | goal/backstory: manifest 읽기 제거, Final Answer 코드 덤프 금지, 도메인 구현 강화 |
| ✅ 11 | `crew_agents/executor.py` | goal/backstory: 날조 금지, 실패 시 return_code=-2 보고 명시 |
| ✅ 12 | `crew_tools/code_tools.py` | `ImportCheckTool`: DLL/OS error → IMPORT_SKIP (코드 문제 아님) |

### 남은 구조적 한계 (수정 불가)

- ReAct 텍스트 파싱 자체 — 프롬프트로 100% 막을 수 없음
- 에이전트 간 컨텍스트 단절 — 문자열 핸드오프의 정보 손실
- `expected_output` pressure — CrewAI가 조기 종료를 유도
- 최대 신뢰도: ~80% (배열 방어로 주요 경로 안정화)

### 수용 기준

다음 두 조건을 만족하면 CrewAI 프로토타입은 "완료"로 간주:
1. `experiment_impl.py`가 연구 주제에 맞는 실제 ML 코드로 작성됨 (sklearn 날조 없음)
2. `python src/main.py`가 실제로 실행되고 `results/result.json`이 디스크에 쓰임

---

## Part 2 — 공통 플랫폼 레이어 (신규)

### 설계 철학: 비교의 공정성

세 프레임워크(CrewAI, LangGraph, AutoGen)를 공정하게 비교하려면  
**안정성 자체가 변수가 되면 안 된다**.  
각 프레임워크의 오케스트레이션 특성이 변수여야 한다.

```
현재 문제:
CrewAI (불안정) vs LangGraph (미완성) vs AutoGen (미구현)
→ 비교 불가: 안정성이 곧 차이

제안 구조:
┌────────────────┬────────────────┬────────────────┐
│   CrewAI 레이어 │ LangGraph 레이어 │  AutoGen 레이어  │  ← 프레임워크 특성 (비교 대상)
│  ReAct 파싱    │  StateGraph    │  GroupChat     │
│  에이전트 역할   │  조건부 엣지    │  대화형 멀티에이전트│
└───────┬────────┴───────┬────────┴───────┬────────┘
        │                │                │
        └────────────────┼────────────────┘
                         ↓
          ┌──────────────────────────────┐
          │   Common Platform Layer      │  ← Claude Code 패턴 (고정)
          │  - tool_loop (while+tool_use)│
          │  - with_retry (exp backoff)  │
          │  - tool_result_budget        │
          │  - token_budget              │
          │  - telemetry                 │
          └──────────────────────────────┘
                         ↓
                  Anthropic API
```

이 구조에서 비교 지표는 순수하게 "각 프레임워크의 오케스트레이션 방식 차이"만 측정한다.

---

### Claude Code에서 차용할 패턴 목록

#### Tier 1 — 필수 (우리 실패의 직접 원인)

| 패턴 | 출처 | 현재 없을 때 증상 |
|------|------|-----------------|
| `tool_use` 루프 (while + API 레벨) | `query.ts` | ReAct 파싱 실패 → 날조 |
| `max_tokens` Resume | `query.ts` | 긴 파일 작성 중 잘림 → 미완성 코드 |
| Tool Result Budgeting | `query.ts` | 긴 stdout → 컨텍스트 폭발 → 조기 종료 |
| Validation Error → Error Block | `toolExecution.ts` | Pydantic 예외 → LLM 관측값 없음 → 날조 |
| Exponential Backoff + Jitter | `withRetry.ts` | 429/529 즉시 실패 → 긴 실험 중단 |

#### Tier 2 — 중요 (장기 실험 안정성)

| 패턴 | 출처 | 효과 |
|------|------|------|
| Token Budget + Diminishing Returns | `tokenBudget.ts` | 반복 수정 루프 감지 → 조기 종료 방지 |
| Circuit Breaker (max 3 failures) | `autoCompact.ts` | repair 무한루프 방지 |
| Tool Concurrency Control | `toolOrchestration.ts` | read-only 병렬 / write 순차 강제 |
| Withholding Mechanism | `query.ts` | transient 에러 → 복구 후 노출 |

#### Tier 3 — 선택 (관찰성)

| 패턴 | 출처 | 효과 |
|------|------|------|
| Query Checkpoints / Telemetry | `query.ts+` | 노드별 토큰·시간 측정 → 비교 지표 |
| Immutable State Pattern | `state.ts` | 상태 오염 방지 (LangGraph는 이미 적용) |

---

### 플랫폼 레이어 구현 계획

#### `platform/tool_loop.py`

핵심: `while(true) + tool_use + max_tokens Resume`

```python
def run_tool_loop(client, model, system, messages, tools, max_turns=30):
    for _ in range(max_turns):
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "max_tokens":           # Resume 패턴
            messages.append({"role": "user", "content": [{
                "type": "text", "text": "Continue directly from where you left off."
            }]})
            continue

        tool_uses = [b for b in response.content if b.type == "tool_use"]
        if not tool_uses:
            return next((b.text for b in response.content if b.type == "text"), ""), messages

        results = []
        for tu in tool_uses:
            result = _safe_execute(tu, tools)            # 예외 → error block 반환
            results.append({"type": "tool_result", "tool_use_id": tu.id,
                            "content": apply_budget(result)})  # 크기 제한
        messages.append({"role": "user", "content": results})
    return "MAX_TURNS_REACHED", messages
```

#### `platform/tool_result_budget.py`

```python
MAX_TOOL_RESULT_CHARS = 4000   # per tool (query.ts 기준)

def apply_budget(result: str, max_chars: int = MAX_TOOL_RESULT_CHARS) -> str:
    if len(result) <= max_chars:
        return result
    half = max_chars // 2
    trimmed = len(result) - max_chars
    return result[:half] + f"\n...[TRUNCATED {trimmed} chars]...\n" + result[-half:]
```

#### `platform/with_retry.py`

```python
BASE_DELAY_MS = 500
MAX_DELAY_MS = 32_000
MAX_RETRIES = 10

def with_retry(fn, *args, **kwargs):
    for attempt in range(MAX_RETRIES):
        try:
            return fn(*args, **kwargs)
        except anthropic.RateLimitError as e:
            if attempt == MAX_RETRIES - 1:
                raise
            base = min(BASE_DELAY_MS * (2 ** attempt), MAX_DELAY_MS) / 1000
            jitter = random.random() * 0.25 * base
            time.sleep(base + jitter)
```

#### `platform/token_budget.py`

```python
COMPLETION_THRESHOLD = 0.90      # 컨텍스트 90% 사용시 경고
DIMINISHING_THRESHOLD = 500      # 연속 turn delta < 500 tokens → 반복 감지

class TokenBudget:
    def check(self, response, context_window: int) -> str:
        delta = response.usage.output_tokens
        self.deltas.append(delta)
        if len(self.deltas) >= 3 and all(d < DIMINISHING_THRESHOLD for d in self.deltas[-3:]):
            return "DIMINISHING"   # → max_turns 조기 종료 트리거
        if response.usage.input_tokens / context_window > COMPLETION_THRESHOLD:
            return "NEAR_LIMIT"    # → compaction 트리거
        return "OK"
```

#### `platform/telemetry.py`

```python
@dataclass
class NodeEvent:
    framework: str       # "crewai" | "langgraph" | "autogen"
    node: str            # "planner" | "coder" | ...
    phase: str           # "enter" | "exit"
    tokens_in: int
    tokens_out: int
    duration_ms: float
    tool_calls: int
    tool_errors: int
```

---

## Part 3 — LangGraph 프로토타입 완성 계획

### 핵심 설계 원칙

**도구 호출이 API 프로토콜 (tool_use 블록) 에 있고, Python이 루프를 제어한다.**  
플랫폼 레이어 위에서 LangGraph는 StateGraph 조건부 엣지만 담당한다.

```
Claude Code:                  현재 CrewAI:
─────────────────                  ──────────────
while (true) {                     LLM 텍스트: "Action: Tool\nAction Input: ..."
  response = api.call()                    ↓
  tool_uses = response.tool_use    텍스트 파서가 추출 시도
  if (!tool_uses) break                    ↓
  for tool in tool_uses:           파싱 실패 가능 → 날조
    result = execute(tool)  ← 보장
    messages.push(result)   ← 보장
}
```

LangGraph 각 노드는 `platform.tool_loop.run_tool_loop()`을 호출한다.

---

### 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                    LangGraph StateGraph                      │
│                                                             │
│  planner → designer → coder ←──────────────────┐           │
│                ↓                                │ repair    │
│            executor → analyzer → writer         │           │
│                ↑           │                    │           │
│                └───────────┘ (should_continue)  │           │
│                            └────────────────────┘ (repair)  │
└─────────────────────────────────────────────────────────────┘

각 노드: Anthropic SDK 직접 사용 (LangChain 아님)
coder/executor 노드: while(true) + tool_use 루프
planner/designer/analyzer: 단일 LLM 호출 + JSON 파싱
```

---

### 기술 스택

| 구성요소 | 선택 | 이유 |
|---------|------|------|
| 그래프 | `langgraph` | 명시적 상태 전이, 조건부 엣지 |
| LLM 클라이언트 | `anthropic` SDK (직접) | native tool_use 블록 필수 |
| 도구 스키마 | `pydantic` BaseModel | 타입 안전 + 명세 자동 생성 |
| 상태 | `TypedDict` (기존 state.py 활용) | 불변 상태 전달 |
| API 서버 | `fastapi` (기존 틀 재사용) | crewai_prototype과 동일 인터페이스 |

**LangChain BaseChatModel 제거**: 현재 `nodes/coder.py`는 `BaseChatModel`을 사용해 단순 LLM 호출만 한다. 이를 Anthropic SDK로 교체해 tool_use 루프를 구현한다.

---

### 단계별 구현 계획

#### Phase 0: 인프라 정리 (½일)

**목표**: 기존 스켈레톤을 정리하고 Anthropic SDK 기반 공통 도구 루프 구현

```
langgraph_prototype/
├── core/
│   ├── tool_loop.py        ← NEW: Claude Code의 queryLoop 패턴
│   ├── tool_registry.py    ← NEW: 도구 등록 + Anthropic schema 변환
│   └── llm_client.py       ← NEW: Anthropic SDK 클라이언트 팩토리
├── workspace/              ← NEW: crewai_prototype/crew_tools 이식
│   ├── workspace_tools.py
│   ├── code_tools.py
│   └── execution_tools.py
```

**`core/tool_loop.py`** — 핵심 구현:

두 가지 핵심 메커니즘이 포함됨:
1. **`stop_reason == "max_tokens"` 감지**: LLM 출력이 잘렸을 때 "Continue directly from where you left off." 주입 → 루프 계속 (Claude Code Resume 패턴)
2. **`WorkspaceWriteTool` append 모드**: LLM이 큰 파일을 여러 turn에 걸쳐 나눠 쓸 수 있도록 `mode: "append"` 지원

```python
def run_tool_loop(
    client: anthropic.Anthropic,
    model: str,
    system: str,
    initial_messages: list[dict],
    tools: list[ToolDef],
    max_turns: int = 30,
) -> tuple[str, list[dict]]:
    """
    Claude Code의 while(true) + tool_use 패턴.
    
    - stop_reason == "max_tokens": "Continue directly from where you left off." 주입 → 루프 계속
    - tool_use 블록이 있으면: 도구 실행 → tool_result 추가 → 계속
    - tool_use 블록이 없으면: Final Answer 반환
    - 결과가 항상 전달됨 (날조 불가)
    """
    messages = list(initial_messages)
    
    for _ in range(max_turns):
        response = client.messages.create(
            model=model,
            system=system,
            messages=messages,
            tools=[t.to_anthropic_schema() for t in tools],
            max_tokens=8192,
        )
        messages.append({"role": "assistant", "content": response.content})
        
        # max_tokens: LLM 출력이 잘린 경우 → Resume 패턴 (Claude Code query.ts)
        if response.stop_reason == "max_tokens":
            messages.append({"role": "user", "content": [{
                "type": "text",
                "text": "Continue directly from where you left off.",
            }]})
            continue
        
        tool_uses = [b for b in response.content if b.type == "tool_use"]
        if not tool_uses:
            # 텍스트만 반환 → 완료
            text = next((b.text for b in response.content if b.type == "text"), "")
            return text, messages
        
        # 도구 실행 — 반드시 결과를 돌려줌
        tool_results = []
        for tu in tool_uses:
            result = _execute_tool(tu, tools)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tu.id,
                "content": result,
            })
        messages.append({"role": "user", "content": tool_results})
    
    return "MAX_TURNS_REACHED", messages
```

---

#### Phase 1: 워크스페이스 도구 이식 (½일)

crewai_prototype의 도구들을 LangGraph용으로 이식.  
차이점: CrewAI `BaseTool` 제거, Anthropic 스키마 직접 노출.

**`WorkspaceWriteTool`에 `mode: "append"` 추가**: 하나의 파일이 단일 턴 토큰 한도를 넘을 경우, LLM이 `mode="append"`로 여러 turn에 걸쳐 나눠 쓸 수 있음. `stop_reason == "max_tokens"` Resume 패턴과 연계됨.

```python
# workspace/workspace_tools.py
class WorkspaceWriteTool:
    name = "workspace_write"
    description = (
        "Write (or append) content to a file in the workspace. "
        "Use mode='write' (default) to create/overwrite, mode='append' to add to existing file. "
        "For files exceeding a single response, call this multiple times with mode='append'."
    )
    
    def to_anthropic_schema(self) -> dict:
        return {
            "name": self.name,
            "description": self.description,
            "input_schema": {
                "type": "object",
                "properties": {
                    "workspace_root": {"type": "string"},
                    "relative_path": {"type": "string"},
                    "content": {"type": "string"},
                    "mode": {
                        "type": "string",
                        "enum": ["write", "append"],
                        "description": "write=overwrite (default), append=add to end of file",
                    },
                },
                "required": ["workspace_root", "relative_path", "content"],
            }
        }
    
    def run(self, workspace_root: str, relative_path: str, content: str, mode: str = "write") -> str:
        path = _resolve(workspace_root, relative_path)
        path.parent.mkdir(parents=True, exist_ok=True)
        if mode == "append":
            with path.open("a", encoding="utf-8") as f:
                f.write(content)
        else:
            path.write_text(content, encoding="utf-8")
        lines = path.read_text(encoding="utf-8").count("\n") + 1
        return f"OK: {mode}d → {relative_path} ({lines} lines total)"
```

이식할 도구 목록:
- `WorkspaceReadTool`, `WorkspaceWriteTool` (append 모드 추가), `WorkspaceListTool`
- `SyntaxCheckTool`, `ImportCheckTool`
- `RunCommandTool`, `ReadResultTool`

---

#### Phase 2: 핵심 노드 구현 (2일)

##### 2-A. Coder 노드 (가장 중요)

현재 상태: LLM 호출 후 코드를 텍스트로 추출 → 상태에 저장 (파일 미작성)  
목표: `run_tool_loop`으로 실제 파일 작성 + 검증

```python
# nodes/coder.py (전면 재작성)
def create_coder_node(client, model, workspace_root):
    tools = [
        WorkspaceReadTool(workspace_root),
        WorkspaceWriteTool(workspace_root),
        WorkspaceListTool(workspace_root),
        SyntaxCheckTool(workspace_root),
        ImportCheckTool(workspace_root),
    ]
    
    def coder_node(state: ResearchState) -> dict:
        design = state["design"]
        mutable_files = state["mutable_files"]  # 번호 목록
        
        system = _build_coder_system(design, state["research_input"])
        initial_messages = [{"role": "user", "content": (
            f"Write these files one at a time:\n{mutable_files}\n"
            "Start with src/artifacts.py (read only), then src/config_schema.py (read only), "
            "then write each mutable file."
        )}]
        
        final_text, messages = run_tool_loop(
            client, model, system, initial_messages, tools
        )
        
        # pre-flight: 파일 실제 존재 확인
        missing = _check_missing_files(workspace_root, mutable_files)
        
        return {
            **update_phase(state, "coding_complete"),
            "code_status": "complete" if not missing else "incomplete",
            "missing_files": missing,
            "coder_messages": messages,  # 디버그용
        }
    
    return coder_node
```

##### 2-B. Executor 노드

현재 상태: 스텁 (구현 없음)  
목표: `RunCommandTool` native 호출 + 실제 결과만 보고

```python
def create_executor_node(client, model, workspace_root):
    tools = [RunCommandTool(), ReadResultTool()]
    
    def executor_node(state: ResearchState) -> dict:
        system = "Execute python src/main.py and report results. Never fabricate."
        messages = [{"role": "user", "content": (
            f"Run the experiment in workspace: {workspace_root}\n"
            "Call RunCommandTool with command='python src/main.py', "
            f"working_dir='{workspace_root}'. "
            "If return_code==0, read results/result.json with ReadResultTool."
        )}]
        
        final_text, _ = run_tool_loop(client, model, system, messages, tools, max_turns=3)
        
        return {
            **update_phase(state, "execution_complete"),
            "execution_output": final_text,
        }
    
    return executor_node
```

##### 2-C. Planner / Designer / Analyzer 노드

단순 LLM 호출 (도구 불필요). 기존 구현을 Anthropic SDK로 교체.

```python
def create_planner_node(client, model):
    def planner_node(state: ResearchState) -> dict:
        response = client.messages.create(
            model=model,
            system=PLANNER_SYSTEM_PROMPT,
            messages=[{"role": "user", "content": _build_planner_prompt(state)}],
            max_tokens=2048,
        )
        plan_json = _extract_json(response.content[0].text)
        return {
            **update_phase(state, "planning_complete"),
            "plan": plan_json,
        }
    return planner_node
```

##### 2-D. Writer 노드

`WriteReportTool`로 Markdown 보고서 작성.  
도구 루프 사용 (파일 작성 확인까지).

---

#### Phase 3: StateGraph 완성 (½일)

기존 `graph/research_graph.py`와 `graph/builder.py`를 실제 동작하도록 완성.

```python
# graph/research_graph.py
from langgraph.graph import StateGraph, END

def build_graph(client, model, workspace_root) -> CompiledGraph:
    builder = StateGraph(ResearchState)
    
    # 노드 등록
    builder.add_node("planner",  create_planner_node(client, model))
    builder.add_node("designer", create_designer_node(client, model))
    builder.add_node("coder",    create_coder_node(client, model, workspace_root))
    builder.add_node("executor", create_executor_node(client, model, workspace_root))
    builder.add_node("analyzer", create_analyzer_node(client, model))
    builder.add_node("writer",   create_writer_node(client, model, workspace_root))
    
    # 선형 경로
    builder.set_entry_point("planner")
    builder.add_edge("planner",  "designer")
    builder.add_edge("designer", "coder")
    builder.add_edge("coder",    "executor")  # pre-flight 실패시 coder로 루프 (조건부)
    builder.add_edge("executor", "analyzer")
    
    # 조건부 엣지: analyzer → writer OR executor (반복)
    builder.add_conditional_edges(
        "analyzer",
        _route_after_analysis,  # should_continue → executor / writer
        {"continue": "executor", "write": "writer", "repair": "coder"},
    )
    builder.add_edge("writer", END)
    
    # pre-flight 실패 조건부 엣지
    builder.add_conditional_edges(
        "coder",
        _route_after_coding,
        {"complete": "executor", "repair": "coder"},
    )
    
    return builder.compile()


def _route_after_analysis(state: ResearchState) -> str:
    debug = state.get("debug_info", {})
    loop_count = debug.get("loop_count", 0)
    max_loops = debug.get("max_loops", 3)
    
    if state.get("meets_target") or loop_count >= max_loops:
        return "write"
    if state.get("repair_needed"):
        return "repair"
    return "continue"
```

**State 보강** (기존 state.py에 추가):
```python
class ResearchState(TypedDict, total=False):
    # 기존 필드 유지 +
    mutable_files: str          # numbered list (coder용)
    missing_files: list[str]    # pre-flight 결과
    repair_needed: bool         # analyzer가 설정
    repair_actions: list[dict]  # 수리 대상 파일/심볼
    workspace_root: str         # 런타임에 설정
    code_status: str            # "complete" | "incomplete"
```

---

#### Phase 4: API 연결 (½일)

crewai_prototype의 FastAPI 구조와 동일한 엔드포인트 노출.  
비교 가능하도록 동일한 요청/응답 포맷 유지.

```
POST /api/research          → 실험 시작
GET  /api/sessions/{run_id} → 상태 조회
GET  /api/events/{run_id}   → 이벤트 스트림 (SSE)
GET  /api/artifacts/{run_id}→ 결과 파일
```

기존 `langgraph_prototype/api/` 스켈레톤을 채운다.

---

#### Phase 5: 비교 테스트 (½일)

동일한 연구 주제로 두 시스템을 동시에 실행해 결과 비교.

**비교 지표**:

| 지표 | CrewAI | LangGraph |
|------|--------|-----------|
| 올바른 도메인 코드 작성률 | ? | ? |
| 도구 호출 성공률 | ? | ? |
| 실제 `result.json` 생성률 | ? | ? |
| Pre-flight 통과까지 repair 횟수 | ? | ? |
| 전체 실행 시간 | ? | ? |

**테스트 케이스**:
1. "ResNet vs ViT on CIFAR-100" — vision classification
2. "XGBoost vs LightGBM on tabular data" — tabular supervised
3. "LSTM vs Transformer for time series" — time series forecasting

---

### 파일 구조 변경 요약

```
research_system/
├── platform/                        ← NEW: 공통 플랫폼 레이어 (모든 프레임워크 공유)
│   ├── tool_loop.py                 ← while + tool_use + max_tokens Resume
│   ├── tool_result_budget.py        ← 도구 결과 크기 제한 (query.ts)
│   ├── with_retry.py                ← exp backoff + jitter (withRetry.ts)
│   ├── token_budget.py              ← 토큰 예산 + 반복 감지 (tokenBudget.ts)
│   └── telemetry.py                 ← 노드별 체크포인트 (query.ts+)
│
├── crewai_prototype/
│   ├── crew_tools/                  ← platform 레이어 활용하도록 교체
│   └── ...
│
├── langgraph_prototype/
│   ├── workspace/                   ← NEW (crewai_prototype 이식 + platform 활용)
│   │   ├── workspace_tools.py
│   │   ├── code_tools.py
│   │   └── execution_tools.py
│   ├── nodes/                       ← 기존 (전면 재작성)
│   │   ├── base.py
│   │   ├── planner.py               ← Anthropic SDK + with_retry
│   │   ├── designer.py              ← Anthropic SDK + with_retry
│   │   ├── coder.py                 ← run_tool_loop + token_budget (핵심)
│   │   ├── executor.py              ← run_tool_loop + tool_result_budget
│   │   ├── analyzer.py              ← Anthropic SDK + with_retry
│   │   └── writer.py                ← run_tool_loop
│   ├── graph/
│   │   ├── state.py                 ← 필드 보강
│   │   ├── research_graph.py        ← 조건부 엣지 완성
│   │   └── builder.py
│   ├── api/routes.py
│   └── main.py
│
└── autogen_prototype/               ← FUTURE: platform 레이어 위에 GroupChat 구현
    └── ...
```

---

### 예상 일정

| Phase | 내용 | 예상 시간 |
|-------|------|----------|
| **P** | **공통 플랫폼 레이어 구현** (tool_loop, with_retry, budget, telemetry) | **1일** |
| 0 | LangGraph 인프라 정리 (platform 연결) | ¼일 |
| 1 | 워크스페이스 도구 이식 (platform 활용) | ¼일 |
| 2 | 핵심 노드 구현 (coder/executor) | 2일 |
| 3 | StateGraph 완성 | ½일 |
| 4 | API 연결 | ½일 |
| 5 | 비교 테스트 (CrewAI vs LangGraph) | ½일 |
| **합계** | | **~5일** |

---

#### Phase 5: 비교 테스트 — 측정 지표 (갱신)

공통 플랫폼 덕분에 안정성 차이가 제거되고, 순수 오케스트레이션 특성만 비교 가능:

| 지표 | 의미 | CrewAI | LangGraph | AutoGen |
|------|------|--------|-----------|---------|
| **도구 호출 성공률** | 프레임워크 파싱 레이어 신뢰도 | ? | ? | ? |
| **컨텍스트 효율** (tokens/task) | 에이전트 간 전달 정보 밀도 | ? | ? | ? |
| **Repair 횟수** | 오케스트레이션의 자기교정 능력 | ? | ? | ? |
| **토큰 낭비율** | 불필요한 반복·재작성 빈도 | ? | ? | ? |
| **실제 result.json 생성률** | 파이프라인 완주율 | ? | ? | ? |
| **전체 실행 시간** | 프레임워크 오버헤드 | ? | ? | ? |

**테스트 케이스**:
1. "ResNet vs ViT on CIFAR-100" — vision classification
2. "XGBoost vs LightGBM on tabular data" — tabular supervised
3. "LSTM vs Transformer for time series" — time series forecasting

---

### 프레임워크 특성 비교 (플랫폼 고정 후)

| | CrewAI | LangGraph | AutoGen |
|---|---|---|---|
| **오케스트레이션** | ReAct 역할극 | StateGraph 조건부 엣지 | GroupChat 대화형 |
| **도구 호출** | ~~텍스트 파싱~~ → platform tool_use | platform tool_use | platform tool_use |
| **에이전트 간 통신** | 문자열 핸드오프 | TypedDict 공유 상태 | 메시지 대화 히스토리 |
| **루프 제어** | LLM 텍스트 (제한적) | Python 명시적 | Python + LLM 협상 |
| **강점** | 역할 기반 분업 자연스러움 | 상태 흐름 명시적 | 다중 에이전트 토론 |
| **약점** | 파싱 오버헤드, 컨텍스트 단절 | 보일러플레이트 많음 | 토큰 비효율 |
| **비교 가능 여부** | ✅ (platform 위) | ✅ (platform 위) | ✅ (FUTURE) |
