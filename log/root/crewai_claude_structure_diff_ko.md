# `crewai_prototype` vs `Claude Code` 구조 검증 문서

> **출처 정리 안내 (2026-08-20)**
>
> 이 문서가 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했습니다(커밋 이력에는 없었음). 그에 따라 **유출 트리를 가리키던
> 파일 경로와 줄번호 인용을 정리**했습니다. 분석 내용 자체는 본인이 쓴 것으로 그대로 둡니다.
>
> 공식 문서: <https://docs.claude.com/en/docs/claude-code>

---

작성일: 2026-04-06

대상 범위:
- 백엔드 후보 1: `crewai_prototype`
- 비교 대상: `Claude Code`
- 현재 프론트엔드 맥락: `research_system_ui`

이 문서는 요청한 3개 역할을 한 문서 안에서 분리해 정리한다.
- 에이전트 1 역할: `crewai_prototype` 구조 파악
- 에이전트 2 역할: `Claude Code` 구조 파악
- 에이전트 3 역할: 두 구조의 사실 검증, 차이점 정리, 현재 실험 맥락에서의 시사점 정리

---

## 0. 먼저 고정해야 하는 사실

### 사실
- 현재 `crewai_prototype`는 단순한 CrewAI 데모가 아니라, FastAPI API 계층, UI 계약 정규화 계층, 스캐폴드 생성 계층, 패치 실행기, 로컬 실행기, 반복 수정 루프를 모두 감싼 "두꺼운 오케스트레이터"다.
- 현재 `Claude Code`은 단일 백엔드가 아니라 CLI 본체, React+Ink 터미널 UI, 다수의 툴 시스템, MCP 클라이언트/서버, 브리지, 서브에이전트, 원격 세션, 별도 `web/` 앱, 별도 `mcp-server/`까지 포함한 큰 플랫폼형 구조다.
- 따라서 두 폴더는 "같은 종류의 백엔드"가 아니다. 하나는 연구 실행 파이프라인 지향, 다른 하나는 범용 코딩 에이전트 플랫폼 지향이다.

### 근거 파일
- `crewai_prototype/main.py`
- `crewai_prototype/crew.py`
- `crewai_prototype/scaffolds/builder.py`
- `cli.tsx`
- `main.tsx`
- `tools.ts`
- `commands.ts`
- `package.js`on`
- (내부 모듈)README.md`

---

## 1. 에이전트 1 역할: `crewai_prototype` 구조 파악

## 1.1 실행 진입점과 런타임 흐름

### 사실
- `crewai_prototype/main.py`는 두 모드를 가진다.
  - CLI 모드: 인자 또는 인터랙티브 입력으로 1회 연구 실행
  - API 모드: FastAPI 서버로 UI와 연동
- API 모드에서 `POST /api/v1/research`가 호출되면:
  1. 요청을 `research_input`으로 변환
  2. `create_default_tools()`로 역할별 도구 묶음을 생성
  3. `ResearchCrew(...)` 인스턴스를 생성
  4. `_active_runs`에 실행 상태를 메모리 등록
  5. `BackgroundTasks`로 `_execute_research()`를 비동기 시작
- 실제 긴 실행은 `_execute_research()`에서 `asyncio.to_thread(crew.run)`으로 워커 스레드에 넘긴다. 즉, FastAPI 이벤트 루프와 연구 실행은 분리되어 있다.
- 로그/상태/UI 스트림은 `main.py` 안에서 함께 처리된다. `GET /api/v1/sessions`, `GET /api/v1/sessions/{run_id}/logs`, `GET /api/v1/research/{run_id}/stream`, `GET /api/v1/research/{run_id}/artifacts/stream`, `GET /api/v1/contract`가 모두 같은 파일 안에 있다.

### 해석
- 실행기, API, UI 계약, 상태 저장이 `main.py`에 집중되어 있어 진입점은 단순하지만 수정 충돌 면적이 크다.
- "백엔드"라기보다 "실행기+API 게이트웨이+세션 상태 정규화기"가 한 파일에 겹쳐 있다.

### 근거 파일
- `crewai_prototype/main.py`
- `crewai_prototype/core/api_contract.py`

## 1.2 내부 모듈 조직

### 사실
- 역할 정의는 `agents/`에 있다.
  - `research_planner.py`
  - `experiment_designer.py`
  - `code_generator.py`
  - `experiment_executor.py`
  - `result_analyzer.py`
  - `paper_writer.py`
- 각 에이전트 팩토리는 CrewAI `Agent(...)`를 생성하며, 역할별 `role`, `goal`, `backstory`를 고정하고 `allow_delegation=False`로 둔다.
- 작업 프롬프트 조립은 `tasks/research_tasks.py`가 담당한다. 여기에는 planning, design, patch plan, coding, analysis, writing task 생성기가 모여 있다.
- 공통 런타임 계층은 `core/`에 있다.
  - `config.py`: `config.yaml`과 `.env` 로딩
  - `llm_factory.py`: 에이전트별 LLM provider/model 매핑
  - `api_contract.py`: UI용 이벤트/세션 모델 및 상태 정규화
  - `project_manifest.py`, `run_contract.py`: 실행 계약
  - `patch_executor.py`: 허용된 파일만 수정하는 패치 적용기
  - `contract_checks.py`: 스캐폴드 계약 검증기
  - `local_context_store.py`: handoff/shared/runtime memory 저장
- 실행 대상 워크스페이스는 `scaffolds/builder.py`가 매 실행마다 `outputs/<run_id>/workspace` 아래에 생성한다.
- 스캐폴드 생성 시 고정 파일과 가변 파일이 분리된다.
  - 고정: `src/main.py`, `src/cli.py`, `src/config_schema.py`, `src/artifacts.py`
  - 가변: `src/experiment_impl.py`, `src/experiment_registry.py`, `src/result_reducer.py`, `src/validation.py` 등
- 도구는 `tools/__init__.py`에서 역할별로 묶인다.
  - planner/designer: ChromaDB 검색/저장
  - coder/writer: ChromaDB + file write
  - executor: E2B + MLflow + file write
  - analyzer: MLflow query + ChromaDB

### 해석
- 이 구조는 "CrewAI가 직접 코드를 만지는 구조"가 아니라, "CrewAI는 계획과 코드 생성만 하고 실제 파일 반영은 오케스트레이터가 통제하는 구조"에 가깝다.
- 겉보기보다 구조가 깊어서, 충돌이 나면 CrewAI 자체 문제인지, 스캐폴드 계약 문제인지, 패치 적용 문제인지, 로컬 실행 문제인지 즉시 분리하기 어렵다.

### 근거 파일
- `crewai_prototype/agents/*.py`
- `crewai_prototype/tasks/research_tasks.py`
- `crewai_prototype/tools/__init__.py`
- `crewai_prototype/scaffolds/builder.py`
- `crewai_prototype/core/config.py`
- `crewai_prototype/core/llm_factory.py`
- `crewai_prototype/core/project_manifest.py`
- `crewai_prototype/core/run_contract.py`
- `crewai_prototype/core/patch_executor.py`
- `crewai_prototype/core/contract_checks.py`

## 1.3 실제 오케스트레이션 흐름

### 사실
- `ResearchCrew.run()`의 상위 흐름은 대략 다음 순서다.
  1. 에이전트 생성
  2. planning task 실행
  3. design task 실행
  4. 반복 수정 루프 시작
  5. iteration마다 coder payload 준비
  6. patch plan 생성
  7. 섹션 단위 coding 생성
  8. 생성된 코드 조각을 `WorkspacePatchExecutor`로 실제 워크스페이스 파일에 반영
  9. 오케스트레이터가 로컬에서 canonical script 실행
  10. analyzer가 execution 결과를 보고 `needs_rework`, `ready_for_report` 등을 판정
  11. 수렴 시 writer 단계로 넘어감
- CrewAI `Crew(...).kickoff()`는 개별 stage 실행마다 사용되지만, 전체 시스템의 주 반복 제어권은 `crew.py`가 갖고 있다.
- coding 단계는 한 번에 전체 파일을 내지 않고, patch target과 section plan으로 나눠 여러 번 생성한다.
- 패치 적용 전 `PatchPlan`을 검증하고, 허용된 mutable path 밖이면 차단한다.
- 패치 적용 후에는 AST parse, compile, import smoke test까지 수행한다.
- 실행 실패가 반복되면 `failure_fingerprint`, `failure_repeat_count`, `blocker_fix_mode`를 handoff state에 반영해 다음 iteration 정책에 영향을 준다.

### 해석
- 이 시스템의 핵심은 CrewAI보다 `crew.py` 쪽이다. 즉 문제의 상당수는 멀티에이전트 프레임워크 충돌이 아니라, 이 위에 덧씌운 반복 패치-실행-검증 상태기계에서 나올 가능성이 높다.
- "프로토콜 충돌과 코드 오류로 원점을 반복"한다는 체감은 구조상 자연스럽다. 계약 파일, 패치 계획, 스캐폴드, 실행 정책, UI 상태 정규화가 서로 영향을 주기 때문이다.

### 근거 파일
- `crewai_prototype/crew.py`
- `crewai_prototype/core/patch_executor.py`
- `crewai_prototype/core/contract_checks.py`
- `crewai_prototype/core/local_context_store.py`

## 1.4 `research_system_ui`와의 연결 흔적

### 사실
- `crewai_prototype/core/api_contract.py`는 주석으로 `research_system_ui/client/src/lib/types.ts`와 정렬된 contract라고 명시한다.
- `main.py`는 UI 친화적 상태 정규화 함수들을 사용해 세션 목록과 로그를 만든다.
- 로그 스트림은 WebSocket이 아니라 SSE(`StreamingResponse`)로 제공된다.
- artifact tail도 SSE로 별도 제공된다.
- `GET /api/v1/contract`가 존재하고, required fields와 스트림 종료 이벤트 형식을 노출한다.

### 사실 검증 중 발견한 추가 사항
- `research_system_ui/client/src/lib/api.ts`는 실제로 `EventSource` 기반 SSE를 사용한다.
- 그런데 `research_system_ui/backend/main.py`와 `research_system_ui/README.md`에는 여전히 `/ws/v1/sessions/{run_id}/stream` WebSocket 설명이 남아 있다.
- 즉, UI 저장소 내부에서도 "예전 WebSocket 백엔드"와 "현재 SSE 계약"이 동시에 남아 있다.

### 해석
- 현재 UI 연동의 핵심 불안정성 중 하나는 백엔드가 하나가 아니라 두 개처럼 보인다는 점이다.
  - `research_system_ui/backend/main.py`: JSONL tail + WebSocket
  - `crewai_prototype/main.py`: richer contract + SSE + artifact endpoints
- 이 상태에서는 실행기 문제와 UI 계약 문제를 쉽게 혼동하게 된다.

### 근거 파일
- `crewai_prototype/core/api_contract.py`
- `crewai_prototype/main.py`
- `research_system_ui/client/src/lib/api.ts`
- `research_system_ui/backend/main.py`
- `research_system_ui/README.md`

## 1.5 구조적으로 충돌을 만들기 쉬운 지점

### 사실 기반 후보
- `crew.py`가 매우 크고, planning/design/coding/patch/execution/analysis/writing을 한 클래스에서 직접 관리한다.
- `_active_runs`는 메모리 기반이며, 로그는 파일 기반이다. 세션 목록은 "로그 디렉터리"와 "메모리 상태"를 병합해서 계산한다.
- coding 단계가 섹션 단위 코드 생성 -> 병합 -> AST 교체 -> import smoke test -> local execution까지 한 iteration 안에 묶여 있다.
- `scaffolds`, `project_manifest`, `run_contract`, `patch_executor`, `contract_checks`가 모두 런타임 필수 경로에 있다.

### 해석
- 구조적 복잡도의 중심은 CrewAI 멀티에이전트보다 "자체적으로 만든 실행 계약 계층"이다.
- 특히 UI 계약과 실행 계약이 한 저장소 안에 섞여 있고, 로컬 실행 상태가 메모리와 파일에 이중 기록되므로 재현성이 흔들리기 쉽다.

### 핵심 확인 파일
- `crewai_prototype/crew.py`
- `crewai_prototype/main.py`
- `crewai_prototype/scaffolds/builder.py`
- `crewai_prototype/core/project_manifest.py`
- `crewai_prototype/core/run_contract.py`
- `crewai_prototype/core/patch_executor.py`

---

## 2. 에이전트 2 역할: `Claude Code` 구조 파악

## 2.1 실행 진입점과 런타임 흐름

### 사실
- `package.json`의 `main`과 `bin`은 `cli.tsx`를 가리킨다.
- `cli.tsx`는 빠른 경로를 먼저 처리한다.
  - `--version`
  - `--dump-system-prompt`
  - `--claude-in-chrome-mcp`
  - `remote-control` / `bridge`
  - `daemon`
  - background session 관련 커맨드
  - environment runner / self-hosted runner
- 빠른 경로에 걸리지 않으면 본체 로딩으로 넘어간다.
- `main.tsx`는 시작 시점에 MDM, keychain prefetch 같은 side effect를 먼저 시작하고, 이후 Commander 기반 CLI 파싱과 React/Ink 렌더링 경로를 로딩한다.
- `init.ts`는 설정, safe env, remote managed settings, policy limits, telemetry, proxy, mTLS, scratchpad, Windows shell 설정 등을 초기화한다.

### 해석
- 이 구조는 처음부터 "멀티 모드 플랫폼" 전제로 설계되어 있다. 단일 `main.py` 서버와는 철학이 다르다.
- 실행 진입점이 분리되어 있어 역할은 명확하지만, 모드가 많아 진입 경로 이해 비용은 높다.

### 근거 파일
- `package.js`on`
- `cli.tsx`
- `main.tsx`
- `init.ts`

## 2.2 내부 모듈 조직

### 사실
- 핵심 모듈은 `src/` 아래에 광범위하게 분리되어 있다.
  - `QueryEngine.ts`: LLM 질의/스트리밍/툴 루프의 중심
  - `Tool.ts`, `tools.ts`, `src/tools/*`: 툴 정의와 레지스트리
  - `commands.ts`, `src/commands/*`: slash command 체계
  - `bridge/`: IDE 연결 브리지
  - `coordinator/`: 멀티에이전트 coordinator mode
  - `tasks/`: background/local/remote agent task
  - `skills/`: skill 로딩/등록
  - `services/mcp/`: MCP client
  - `entrypoints/mcp.ts`: MCP server mode
  - `state/`, `context/`, `hooks/`, `components/`: React/Ink 앱 레이어
  - `remote/`: remote session
  - `memdir/`: `CLAUDE.md` 기반 메모리
- `web/`는 별도 Next.js 앱이다.
- `mcp-server/`는 또 다른 별도 프로젝트이며, Claude Code 소스를 탐색하는 독립 MCP 서버다.

### 해석
- `Claude Code`은 "하나의 앱"이 아니라 CLI 코어를 중심으로 여러 런타임을 품은 mono-repo형 구조에 가깝다.
- UI, MCP, bridge, tasks가 코어와 분리되어 있어 확장점은 풍부하지만, 전체를 한 번에 옮겨 쓰기에는 너무 크다.

### 근거 파일
- `tools.ts`
- `commands.ts`
- `mcp.ts`
- `package.js`on`
- (내부 모듈)README.md`
- (내부 모듈)architecture.md`
- (내부 모듈)subsystems.md`

## 2.3 툴/프로토콜/에이전트 오케스트레이션 방식

### 사실
- `tools.ts`는 `AgentTool`, `BashTool`, `FileReadTool`, `FileEditTool`, `FileWriteTool`, `WebFetchTool`, `WebSearchTool`, `SkillTool`, MCP 리소스 툴, Task 툴 등을 중앙 등록한다.
- 다수의 툴이 feature flag와 환경 조건에 따라 조건부 포함된다.
- `commands.ts`는 slash command들을 중앙 등록한다. bridge, mcp, plugin, tasks, permissions, review, plan, agents 등 명령이 광범위하다.
- `QueryEngine.ts`는 메시지 스트림, 툴 사용, retry, transcript 기록, structured output, compact boundary, tool result 반영 등을 담당한다.
- `AgentTool.tsx`는 서브에이전트 실행을 정식 툴로 제공한다.
  - 입력 스키마에 `description`, `prompt`, `subagent_type`, `run_in_background`, `mode`, `isolation`, `cwd` 등이 있다.
  - local agent, remote agent, teammate, worktree isolation을 고려한다.
- `coordinatorMode.ts`는 coordinator mode일 때 어떤 툴을 worker에게 허용할지, worker 결과를 어떤 XML task notification으로 받을지, 어떤 식으로 병렬화할지 시스템 프롬프트 차원에서 규정한다.
- `mcp.ts`는 Claude Code 툴들을 MCP 서버로 다시 노출한다.

### 해석
- 이 구조의 핵심은 "에이전트도 툴"이라는 점이다. 서브에이전트, task, MCP가 모두 같은 툴 기반 계약 위에 놓인다.
- `crewai_prototype`가 iteration state machine 중심이라면, `Claude Code`은 tool protocol 중심이라고 보는 편이 더 정확하다.

### 근거 파일
- `tools.ts`
- `commands.ts`
- `QueryEngine.ts`
- `AgentTool.tsx`
- `coordinatorMode.ts`
- `mcp.ts`

## 2.4 안정성을 높이는 설계 요소

### 사실
- 초기화 로직이 `entrypoints/init.ts`로 분리되어 있다.
- 툴과 명령은 중앙 레지스트리에서 관리된다.
- 다수의 스키마와 입력 검증이 존재한다.
- 기능 단위 feature flag와 lazy import가 많다.
- permissions, bridge, MCP, tasks, skills가 서브시스템별로 분리되어 문서화돼 있다.
- `QueryEngine.ts`는 retry, transcript, tool-loop, compaction을 한 곳에서 관리한다.

### 해석
- 이 구조는 "기능이 많아도 각 기능의 경계가 비교적 명시적"이라는 장점이 있다.
- `crewai_prototype`처럼 한 클래스가 계획부터 결과 작성까지 다 장악하는 방식보다 책임 분리가 더 강하다.

### 근거 파일
- `init.ts`
- `tools.ts`
- `commands.ts`
- (내부 모듈)architecture.md`
- (내부 모듈)subsystems.md`

## 2.5 복잡도를 높이는 지점

### 사실
- feature flag가 매우 많다.
- CLI, remote bridge, daemon, MCP server, web, remote session, teammate/swarm, plugin, skill, memory까지 범위가 넓다.
- `main.tsx`, `QueryEngine.ts`, `Tool.ts`, `commands.ts` 같은 대형 파일이 존재한다.

### 해석
- 구조는 분할돼 있지만 시스템 전체 복잡도는 매우 높다.
- 즉, "구조가 좋아 보여서 바로 대체 백엔드로 넣는 것"과 "현재 연구 시스템 문제를 실제로 줄이는 것"은 별개다.

### 핵심 확인 파일
- `main.tsx`
- `QueryEngine.ts`
- `commands.ts`
- `tools.ts`
- `AgentTool.tsx`

---

## 3. 에이전트 3 역할: 사실 검증과 차이점 정리

## 3.1 차이점 표

| 항목 | `crewai_prototype` | `Claude Code` | 검증 요약 |
|---|---|---|---|
| 시스템 목적 | 연구 실행 파이프라인 | 범용 코딩 에이전트 플랫폼 | 목적 자체가 다름 |
| 주 실행 코어 | `ResearchCrew.run()` 반복 루프 | `QueryEngine` + tool loop | 상태기계 vs 툴 프로토콜 |
| 진입점 구조 | `main.py` 한 파일에 CLI/API 공존 | `entrypoints/cli.tsx` + `main.tsx` + `init.ts` | Claude 쪽이 더 분리됨 |
| UI 연결 | FastAPI + SSE + 세션/아티팩트 API | 터미널 UI 기본, 별도 `web/` 존재 | drop-in 교체 불가 |
| 서브에이전트 모델 | CrewAI 역할 에이전트 고정 | `AgentTool`로 동적 spawn | Claude 쪽이 더 일반화됨 |
| 실행 대상 코드 | 매 run마다 scaffold workspace 생성 | 현재 프로젝트/세션 중심 작업 | 작업 단위가 다름 |
| 코드 수정 안전장치 | manifest/run_contract/patch_executor | permissions/worktree/isolation/tool schemas | 안전장치 종류가 다름 |
| 상태 저장 | `_active_runs` + JSONL 로그 + outputs | AppState + transcript + task/session infra | Crew 쪽은 UI 세션 편향, Claude 쪽은 대화/작업 편향 |
| 확장 방식 | profile/scaffold/tool 추가 | tool/command/plugin/skill/MCP 추가 | Claude 쪽 확장면이 훨씬 넓음 |
| 원격/브리지 | 사실상 없음 | bridge, remote, daemon, mcp, web | Claude 쪽이 훨씬 큼 |

## 3.2 중요한 사실 검증 결과

### 사실 1
- `crewai_prototype`는 이미 `research_system_ui`를 의식한 API contract 계층을 갖고 있다.
- 즉, 백엔드 교체 전에 먼저 정리할 대상은 "CrewAI 사용 여부"보다 "중복 백엔드와 UI 계약 드리프트"일 가능성이 높다.

### 사실 2
- `research_system_ui` 내부에는 두 종류의 백엔드 흔적이 공존한다.
  - 현재 프런트 API 호출 코드: SSE 기반
  - 별도 `backend/main.py`와 README: WebSocket 기반
- 이 차이는 실제 프로토콜 충돌의 직접 원인이 될 수 있다.

### 사실 3
- (내부 모듈)는 `NEXT_PUBLIC_API_URL` 기본값 `http://localhost:3001`의 별도 백엔드를 기대한다.
- 따라서 이것은 `research_system_ui`와 바로 호환되는 프론트가 아니다.

### 사실 4
- `Claude Code`은 연구 실험 scaffold, result schema, validation contract, experiment execution loop를 기본 제공하지 않는다.
- 반대로 `crewai_prototype`는 바로 그 부분이 핵심이다.

## 3.3 현재 문제 맥락에서의 해석

### 해석
- 지금의 병목은 "멀티 에이전트 프레임워크가 CrewAI라서"라기보다, 아래 세 층이 한 번에 흔들리는 점에 더 가깝다.
  1. 연구 실행 계약층
  2. UI 세션/로그 프로토콜층
  3. 코드 생성 후 실제 반영/실행/검증 상태기계
- `Claude Code`은 이 중 2번과 범용 orchestration 쪽에서 참고할 부분이 많지만, 1번을 대체해주지는 않는다.
- 즉 "현재 백엔드를 버리고 Claude Code를 그대로 넣는 것"은 대안이 아니라, 종류가 다른 대형 시스템을 새로 붙이는 일이다.

## 3.4 즉시 결론

### 사실 기반 결론
- `Claude Code`은 직접적인 대체 백엔드가 아니다.
- `crewai_prototype`의 직접적 구조 문제는 다음 셋이다.
  - API/UI 계약과 실행기 책임이 강하게 결합됨
  - `research_system_ui` 저장소 자체에도 구형 WebSocket 백엔드 흔적이 남아 있음
  - `crew.py`가 계획, 패치, 실행, 분석, 보고를 모두 흡수한 두꺼운 제어층임

### 실무적 결론
- 다음 단계는 "대체"보다 "분리"가 맞다.
  1. `research_system_ui`가 따를 단일 프로토콜을 먼저 고정
  2. `crewai_prototype`에서 UI 계층을 분리
  3. 반복 수정 루프를 별도 실행 서비스로 축소
  4. 그 뒤에야 Claude Code 계열의 툴/에이전트/태스크 패턴을 필요한 만큼 이식하는 것이 맞다

---

## 4. 지금 시점의 권장 판단

### 권장하지 않는 선택
- `Claude Code`을 그대로 현재 `research_system_ui`의 백엔드로 교체
- `crewai_prototype`와 `research_system_ui/backend/main.py`를 동시에 유지
- UI 프로토콜을 정하지 않은 상태에서 멀티에이전트 구조를 더 키우기

### 더 현실적인 선택
- `crewai_prototype`를 "연구 실행 엔진"으로 축소
- `research_system_ui`는 `crewai_prototype/main.py`의 SSE 계약만 따르도록 정리
- `Claude Code`에서는 아래 패턴만 선별적으로 참고
  - 중앙 툴 레지스트리
  - 서브에이전트 spawn/notification 프로토콜
  - task 기반 비동기 작업 추적
  - permission/isolation/worktree 개념
  - UI와 코어를 나누는 진입점 구조

---

## 5. 핵심 근거 파일 목록

### `crewai_prototype`
- `crewai_prototype/main.py`
- `crewai_prototype/crew.py`
- `crewai_prototype/core/api_contract.py`
- `crewai_prototype/core/config.py`
- `crewai_prototype/core/llm_factory.py`
- `crewai_prototype/core/project_manifest.py`
- `crewai_prototype/core/run_contract.py`
- `crewai_prototype/core/patch_executor.py`
- `crewai_prototype/core/contract_checks.py`
- `crewai_prototype/scaffolds/builder.py`
- `crewai_prototype/tasks/research_tasks.py`
- `crewai_prototype/tools/__init__.py`
- `crewai_prototype/agents/code_generator.py`
- `crewai_prototype/agents/experiment_designer.py`
- `crewai_prototype/agents/experiment_executor.py`
- `crewai_prototype/agents/paper_writer.py`
- `crewai_prototype/agents/research_planner.py`
- `crewai_prototype/agents/result_analyzer.py`

### `Claude Code`
- `package.js`on`
- `cli.tsx`
- `main.tsx`
- `init.ts`
- `mcp.ts`
- `QueryEngine.ts`
- `tools.ts`
- `commands.ts`
- `coordinatorMode.ts`
- `AgentTool.tsx`
- `package.js`on`
- `route.ts`
- (내부 모듈)README.md`
- (내부 모듈)architecture.md`
- (내부 모듈)subsystems.md`

### `research_system_ui`
- `research_system_ui/client/src/lib/api.ts`
- `research_system_ui/client/src/lib/types.ts`
- `research_system_ui/backend/main.py`
- `research_system_ui/README.md`

---

## 6. 다음 단계 제안

이 문서 기준으로 바로 이어질 작업 우선순위는 다음이 가장 합리적이다.

1. `research_system_ui`가 실제로 따를 단일 백엔드 계약을 확정한다.
2. `crewai_prototype/main.py`에서 UI API 계층과 연구 실행 계층을 분리한다.
3. `crew.py` 반복 루프를 별도 서비스 또는 별도 모듈 경계로 쪼갠다.
4. 그 다음에야 `Claude Code`에서 서브에이전트/task/tool registry 패턴을 차용할지 결정한다.

이 순서를 거꾸로 하면, 프레임워크를 바꿔도 현재의 프로토콜 충돌과 책임 혼재가 그대로 재발할 가능성이 높다.
