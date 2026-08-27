# 제작 노트 — AI 에이전트 프레임워크 비교 연구 시스템

이 문서는 세 가지 AI 에이전트 프레임워크(CrewAI, AutoGen, LangGraph)로 동일한 연구 워크플로우를 구현하는 과정에서 마주친 문제들과 해결 방법을 기록한다. 동일한 구조를 만들려는 개발자들이 같은 함정에 빠지지 않도록 작성했다.

워크플로우: `planner → designer → coder → executor → analyzer → writer`  
각 에이전트가 이전 에이전트의 결과물을 받아 다음 단계로 전달하는 파이프라인.

---

## 목차

1. [세 프레임워크 공통 문제](#1-세-프레임워크-공통-문제)
2. [CrewAI 특유 문제](#2-crewai-특유-문제)
3. [AutoGen 특유 문제](#3-autogen-특유-문제)
4. [LangGraph 특유 문제](#4-langgraph-특유-문제)
5. [설계 결정 및 교훈](#5-설계-결정-및-교훈)

---

## 1. 세 프레임워크 공통 문제

### 1-1. 파일 쓰기: LLM이 도구를 호출할 것이라고 가정하면 안 된다

**증상**: 코더 에이전트가 성공적으로 실행됐다고 나오는데, workspace 디렉토리에 파일이 없다.

**원인**: 모든 프레임워크에서 "파일 쓰기 도구를 호출하라"는 지시를 받은 LLM이 코드를 도구 호출로 실행하는 대신 텍스트 응답에 포함해서 반환하는 경우가 있다. 특히:
- 컨텍스트에 이미 코드처럼 보이는 텍스트(설계 명세, 의사코드)가 있으면 LLM이 "이미 코드가 있다"고 판단해 텍스트로 반환함
- 프롬프트 포맷이 실제 API 동작 방식과 맞지 않으면(아래 CrewAI 절 참고) 도구 호출이 발생하지 않음

**권장 해결책**: 파일 I/O를 LLM 의지에 맡기지 말 것.

```python
# 나쁜 구조: LLM이 도구를 호출해주길 기대
crew.kickoff()  # LLM이 WorkspaceWriteTool을 호출하지 않으면 파일 없음

# 좋은 구조: LLM은 내용 생성, Python이 파일 쓰기
content = llm.call([{"role": "user", "content": generate_prompt}])
Path(workspace_root, file_path).write_text(strip_fences(content))
```

LLM에게는 순수하게 "이 파일의 Python 소스 코드를 작성해라"만 시키고, 실제 파일 시스템 작업은 오케스트레이터가 직접 수행하는 것이 가장 안정적이다.

---

### 1-2. LLM 출력 토큰 한도: 응답이 중간에 잘린다

**증상**: 생성된 파일이 불완전하게 잘려있거나, 함수가 중간에 끊긴다.

**원인**: LLM의 최대 출력 토큰을 초과하면 응답이 잘린다. 한 파일에 20개 함수를 담으면 분명히 일어난다.

**해결책**:
- 파일 한 개의 역할(responsibility)을 작게 유지한다. 한 파일 = 한 가지 책임.
- 설계 단계에서 파일을 작게 쪼개는 것이 구현 단계에서 repair를 반복하는 것보다 훨씬 싸다.
- `max_tokens` 설정을 확인하라. CrewAI는 OpenAI 모델에서 `max_tokens`를 `max_completion_tokens`로 변환하는 과정에서 400 에러가 나는 버그가 있었다 (버전에 따라 다름).

---

### 1-3. 의존성 컨텍스트: LLM이 다른 파일의 API를 모른다

**증상**: `from src.models import build_model`을 호출하는데 실제 `models.py`에는 `create_model`로 이름이 다르다. Import 에러가 계속 발생한다.

**원인**: 코더가 파일 A를 쓸 때, 파일 A가 import하는 파일 B의 실제 내용을 모른다. 설계 문서에 "exports: [build_model]"이라고 써있어도, 이전 단계에서 실제로 다른 이름으로 구현했을 수 있다.

**해결책**: 파일을 생성 순서대로 쓰고(stage 1 → stage 2), 의존성 파일의 실제 소스를 프롬프트에 포함한다.

```python
# 의존성 파일 내용을 프롬프트에 포함
deps_context = ""
for dep_path in file_spec.imports_from:
    if Path(workspace_root, dep_path).exists():
        content = Path(workspace_root, dep_path).read_text()
        deps_context += f"# === {dep_path} ===\n{content}\n\n"

prompt = f"""...
Dependency files (already written; your imports must match):
{deps_context}
"""
```

단, 의존성 파일이 많으면 컨텍스트가 폭발하므로 파일당 상한을 설정한다 (예: 6,000자).

---

### 1-4. 에이전트 간 핸드오프: 긴 텍스트를 그대로 넘기지 말 것

**증상**: 다음 에이전트가 이전 에이전트의 출력을 오해하거나, 컨텍스트가 너무 커서 토큰 한도 초과.

**원인**: 실행 로그, 긴 분석 결과, 설계 문서 전체를 다음 에이전트 프롬프트에 직접 붙여넣으면 컨텍스트가 빠르게 폭발한다.

**해결책**: JSON 구조체로 핵심 정보만 넘기고, 긴 텍스트는 파일로 저장한 뒤 경로만 전달한다.

```python
# 나쁜 패턴
next_agent_input = {
    "previous_output": very_long_log_string  # 수만 자
}

# 좋은 패턴
log_path = save_to_file(very_long_log_string)
next_agent_input = {
    "log_path": log_path,          # 경로만
    "summary": {"accuracy": 0.85}  # 핵심 수치만
}
```

---

### 1-5. 재현 가능성: seed를 고정하지 않으면 결과가 매번 달라진다

**증상**: 같은 설정으로 두 번 실행했는데 결과가 달라진다.

**해결책**:
- ML 코드에서는 PyTorch/NumPy/Python random seed를 모두 고정한다.
- LLM 호출에서는 `temperature=0.0`을 사용한다 (코드 생성, 분석 에이전트).
- 데이터 분할(train/val/test)은 seed 고정 후 인덱스를 파일로 저장해 다음 실행에서도 동일한 분할을 사용한다.

---

### 1-6. 구문 검사는 쓰기 직후 바로 실행해야 한다

**증상**: 20개 파일을 모두 생성한 뒤 실행하면 첫 번째 파일에 구문 에러가 있어서 전체가 실패한다.

**해결책**: 파일을 하나 쓸 때마다 바로 `ast.parse()`로 구문을 검사하고, 실패하면 즉시 수정한다. 전체를 생성한 뒤 검사하면 repair 비용이 훨씬 크다.

---

## 2. CrewAI 특유 문제

### 2-1. ReAct 포맷 vs Native Function Calling: 버전에 따라 다르다

**핵심 혼란**: CrewAI 문서와 예제 코드 상당수가 ReAct 포맷(`Action: ToolName\nAction Input: {...}`)을 기준으로 작성됐다. 그런데 CrewAI 0.90+ 이후 내부적으로 OpenAI의 native function calling (JSON Schema 기반)으로 전환됐다.

**실제로 발생한 문제**:
- Task description에 `Action: 1. Call WorkspaceWriteTool\nAction Input: {...}` 형식을 넣으면 LLM이 이것을 "출력해야 할 텍스트"로 인식해서 그대로 텍스트로 반환함. 도구가 호출되지 않음.
- `STEP 0 / STEP 1` 같은 ASCII 구조 배너를 넣으면 LLM이 "명세서를 분석하는 모드"로 진입해서 도구 호출 없이 분석 텍스트만 반환함.

**해결책**:
```python
# 나쁜 예 (ReAct 포맷, native function calling 환경에서 역효과)
task_description = """
STEP 0: Read the designer context
Action: WorkspaceReadTool
Action Input: {"relative_path": "context/designer_output.json"}

STEP 1: Write the file
Action: WorkspaceWriteTool
Action Input: {"relative_path": "src/model.py", "content": "..."}
"""

# 좋은 예 (직접 지시어)
task_description = """
You have real file-system tools. Use them now.
Call WorkspaceWriteTool with the complete implementation of src/model.py.
"""
```

그러나 이 방법으로도 LLM이 도구를 안 부르는 경우가 있었다 (2-2 참고). 결국 에이전트를 제거하는 것이 근본 해결책이었다.

---

### 2-2. WorkspaceWriteTool 미호출: 근본 해결은 에이전트 제거

**현상**: `Crew.kickoff()` 완료 후 파일이 없다. `_run_checks()`가 "File not found" 반환.

**원인 분석**:

1. CrewAI native function calling 환경에서 LLM은 도구를 호출할지, 텍스트를 반환할지 스스로 결정한다. `tool_choice="required"`를 설정하지 않으면 LLM이 텍스트 반환을 선택할 수 있다.

2. Designer 출력 JSON(파일 명세 20개+)이 크면 컨텍스트에 포함될 때 도구 설명이 잘려나가거나, LLM이 "설계 내용이 이미 있으니 텍스트로 반환"이라고 판단한다.

3. 어떤 프롬프트 포맷을 써도 LLM 버전 업데이트나 모델 변경에 따라 동작이 달라질 수 있다.

**최종 해결책**: Phase 2 코딩에서 CrewAI Agent/Crew/Task를 완전히 제거하고 직접 LLM 호출로 대체.

```python
# crewai_prototype/phases/phase2_coding.py

def _generate_content(file_spec, workspace_root, stack_rule, extra_context, llm):
    deps = _build_dep_context(file_spec.imports_from, workspace_root)
    prompt = build_write_prompt(file_spec, stack_rule, extra_context, deps)
    raw = llm.call([{"role": "user", "content": prompt}])
    return _strip_fences(raw)

def _repair_loop(file_spec, ...):
    content = _generate_content(...)  # LLM이 텍스트 반환
    _write_to_disk(file_path, workspace_root, content)  # Python이 직접 씀
    check = _run_checks(file_path, workspace_root)      # 항상 검사
    ...
```

이 방식은 "LLM이 도구를 호출해주길 기대"하는 구조를 없앤다. 파일 쓰기 실패가 구조적으로 불가능해진다.

---

### 2-3. LLM이 JSON 배열을 단일 dict 도구에 전달한다

**증상**: `"the Action Input is not a valid key, value dictionary"` 에러.

**원인**: GPT-4/5 계열 모델이 여러 도구 호출을 배치(batch)로 묶으려는 경향이 있다. 도구의 schema가 단일 dict를 기대하는데 `[{...}, {...}]` 배열을 전달한다.

**해결책 두 가지**:

1. `parallel_tool_calls=False`를 LLM에 설정한다 (OpenAI 전용):
```python
llm_kwargs["parallel_tool_calls"] = False
```

2. 도구의 `args_schema`에 배열 자동 언래핑 로직을 추가한다:
```python
class _ArraySafeModel(BaseModel):
    @model_validator(mode="before")
    @classmethod
    def _unwrap_array(cls, data):
        if isinstance(data, list) and data and isinstance(data[0], dict):
            return data[0]
        return data
```

---

### 2-4. `expected_output` 필드가 조기 종료를 유발한다

**증상**: 에이전트가 첫 번째 도구 호출 후 "DONE"을 출력하고 멈춘다. 나머지 파일들이 생성되지 않는다.

**원인**: CrewAI Task의 `expected_output` 필드 값이 LLM 응답에 등장하면 CrewAI가 태스크 완료로 판단한다.

**해결책**: `expected_output`을 최대한 구체적으로 설정하거나, 완료 신호를 파일 존재 여부로 코드에서 직접 검증한다.

---

### 2-5. 설계 파일 수가 많아지면 컨텍스트가 폭발한다

**발생 조건**: Designer가 20개 이상의 파일 명세를 생성하면 JSON 크기가 30KB를 넘는다. `context_char_budget: 24000` (config.yaml 기본값)을 초과해서 Coder의 컨텍스트에서 도구 설명이 잘린다.

**해결책**:
- 설계 단계에서 파일 수에 상한을 설정한다 (경험상 15개 이하 권장).
- 각 파일의 코딩 태스크에는 해당 파일의 명세만 전달하고 전체 Designer JSON을 넘기지 않는다.

---

## 3. AutoGen 특유 문제

### 3-1. 정규식 기반 코드 추출은 취약하다

**현상**: Coder 에이전트가 코드를 생성했는데 파일이 없다. 로그를 보면 코드가 메시지 텍스트에 있다.

**원인**: AutoGen에서는 Coder 에이전트가 도구를 호출하는 대신 ` ```python ` 코드 블록을 메시지로 출력하고, 오케스트레이터가 정규식으로 이를 추출해서 저장하는 패턴을 쓰기 쉽다. 이 패턴은 LLM이 블록 형식을 살짝만 바꿔도 파싱이 실패한다.

**해결책**: 직접 LLM 호출 + Python 파일 쓰기 패턴(1-1 참고)을 AutoGen에도 적용한다. `AssistantAgent`가 도구를 반드시 호출하도록 `tool_choice` 설정을 확인한다.

---

### 3-2. SelectorGroupChat 라우팅이 코드 블록 존재에 의존한다

**현상**: Coder → Critic 라우팅이 실패하거나 무한 루프.

**원인**: `custom_selector_func`가 메시지에 ` ```python ` 블록이 있는지로 에이전트를 선택할 때, LLM이 블록 없이 코드를 설명하는 텍스트를 반환하면 라우팅이 오작동한다.

**해결책**: 라우팅 기준을 메시지 텍스트 파싱이 아닌 명시적 상태(예: 파일 존재 여부, 구조화된 JSON 출력)로 바꾼다.

---

### 3-3. 모든 에이전트가 전체 대화 히스토리를 공유한다

**현상**: 대화가 길어질수록 토큰 비용이 급증하고, 초기 컨텍스트 윈도우 제한에 걸린다.

**원인**: AutoGen의 `SelectorGroupChat`은 모든 에이전트가 전체 메시지 히스토리를 본다. 파이프라인이 길수록 히스토리가 쌓인다.

**해결책**: 단계가 완료될 때마다 히스토리를 요약하거나, 다음 단계에는 요약본만 포함한 새 채팅을 시작한다.

---

## 4. LangGraph 특유 문제

### 4-1. 플랫폼 레이어 의존성: tool_loop가 없으면 폴백이 침묵으로 실패한다

**현상**: Coder 노드가 실행됐는데 파일이 없고 로그에 "rsp not available"만 남는다.

**원인**: `langgraph_prototype/nodes/coder.py`가 `rsp.tool_loop`라는 외부 플랫폼 레이어에 의존한다. 이 모듈이 없으면 폴백 경로(`final_text, history = "rsp not available", []`)로 빠지고 아무 파일도 쓰지 않는다.

**해결책**: 외부 의존성 없이 Anthropic API를 직접 호출하는 tool loop를 직접 구현하거나, 1-1에서 설명한 직접 LLM 호출 패턴을 사용한다.

---

### 4-2. Executor 시뮬레이션 폴백이 너무 관대하다

**현상**: 실험이 성공했다고 나오는데 결과 파일을 열면 랜덤 숫자가 채워져 있다.

**원인**: Docker 환경이 없을 때 `_simulate_execution()`이 실행된다. 이 함수는 랜덤하게 "성공" 메트릭을 생성한다. 실패가 명확하게 보고되지 않는다.

**해결책**: 시뮬레이션 폴백을 제거하거나, 시뮬레이션임을 결과에 명시적으로 표시해서 분석 에이전트가 실제 결과와 구분하게 한다.

```python
# 나쁜 패턴
def _simulate_execution():
    return {"accuracy": random.uniform(0.7, 0.9), "status": "success"}  # 거짓 성공

# 좋은 패턴
def _execute_or_fail():
    if not docker_available():
        raise RuntimeError("Docker not available. Cannot execute experiment.")
    return _run_in_docker()
```

---

### 4-3. ResearchState 필드 타입 검증이 없다

**현상**: 노드 A가 `state["design"]`에 dict를 넣었는데 노드 B가 str으로 처리하려다 TypeError.

**원인**: LangGraph의 `TypedDict` 기반 상태는 런타임에 타입을 강제하지 않는다. 노드 간에 필드 형식에 대한 암묵적 합의가 깨지면 디버깅이 어렵다.

**해결책**: Pydantic 모델을 상태 타입으로 사용하거나, 각 노드 진입 시점에 명시적 타입 검증을 추가한다.

---

## 5. 설계 결정 및 교훈

### 5-1. 파일 쓰기 아키텍처 비교

| 접근 방식 | 안정성 | 유연성 | 권장 여부 |
|-----------|--------|--------|----------|
| LLM이 쓰기 도구 호출 | 낮음 (LLM이 안 부를 수 있음) | 높음 | **비권장** |
| 정규식으로 코드 블록 추출 | 낮음 (포맷 변경에 취약) | 중간 | **비권장** |
| **LLM 직접 호출 + Python 파일 쓰기** | **높음** | 중간 | **권장** |
| 구조화된 출력(JSON) + Python 파일 쓰기 | 높음 | 높음 | 권장 (복잡도 증가) |

### 5-2. 어떤 프레임워크가 이 파이프라인에 더 적합한가

세 프레임워크 모두 "멀티 에이전트가 파일을 생성하고 실험을 실행하는" 워크플로우를 위해 설계되지 않았다. 각 프레임워크의 강점이 다른 곳에 있다.

- **CrewAI**: 역할 기반 협업에 최적화. 파일 생성 파이프라인은 에이전트 레이어를 우회하는 것이 안정적이다.
- **AutoGen**: 대화형 코드 생성(REPL 스타일)에 적합. 단방향 파이프라인보다 반복적 대화가 맞는 환경에서 더 자연스럽다.
- **LangGraph**: 상태 머신 기반 파이프라인에 적합. 단계 간 상태 관리가 명확하지만, LLM 호출 레이어를 직접 제어할 수 없는 추상화가 걸림돌이 될 수 있다.

### 5-3. 프롬프트 엔지니어링은 임시방편이다

이 프로젝트에서 가장 많은 시간을 쓴 부분은 "LLM이 도구를 호출하게 만드는 프롬프트"를 찾는 것이었다. 결론: 프롬프트로 LLM 동작을 100% 제어하려는 시도는 모델 버전이 바뀌면 무너진다.

더 근본적인 접근: **LLM이 잘못된 동작을 할 수 없는 구조를 만들어라.** LLM의 역할을 "텍스트 생성"으로 제한하고, 파일 I/O·상태 관리·검증은 오케스트레이터 Python 코드가 책임지게 하라.

### 5-4. 수정 루프(Repair Loop)는 필수다

LLM이 생성한 코드는 반드시 구문 에러나 import 에러가 있다고 가정하고 설계하라. 첫 번째 시도에 항상 성공하는 LLM은 없다. 빠른 검사(ast.parse) + 자동 수정 루프 + 사용자 에스컬레이션의 3단계 구조가 실용적이다.

---

## 버전 정보

| 프레임워크 | 패키지 | 버전 |
|-----------|--------|------|
| CrewAI | `crewai[tools]` | `>=0.203.0,<1.0.0` |
| AutoGen | `autogen-agentchat` | `==0.4.7` |
| LangGraph | `langgraph` | `>=0.2.0` |
| 공통 | `openai` | `>=1.30.0` |
| 공통 | `anthropic` | `>=0.25.0` |

---

*최종 업데이트: 2026-05-21*
