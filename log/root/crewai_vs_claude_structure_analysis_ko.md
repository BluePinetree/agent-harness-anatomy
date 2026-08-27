# 자동 연구 시스템 구조 검증 보고서

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
대상:
- `crewai_prototype`
- `Claude Code`
- 연동 참고: `research_system_ui`

## 1. 범위와 검증 기준

이 문서는 로컬 워크스페이스에 있는 코드만 기준으로 작성했다. 외부 주장, 저장소 출처, "유출본"의 진위는 여기서 검증하지 않았다.  
검증 기준은 다음 3가지다.

1. 실제 진입점과 런타임 흐름이 코드에 존재하는가
2. 에이전트, 태스크, 도구, 상태, 출력 구조가 코드 수준에서 어떻게 조직되는가
3. 두 구조가 어떤 점에서 본질적으로 다르고, 현재 `research_system_ui`와의 궁합에 어떤 영향을 주는가

## 2. 분석 트랙 요약

### 트랙 A. `crewai_prototype` 구조 파악

#### 사실

- 메인 진입점은 Python 단일 파일 `crewai_prototype/main.py`에 집중되어 있다. CLI와 FastAPI 서버를 한 파일에서 모두 처리한다.
  - 근거: `crewai_prototype/main.py:48`, `crewai_prototype/main.py:262`, `crewai_prototype/main.py:1251`
- 실행 모드는 `cli`와 `api` 두 가지다.
  - 근거: `crewai_prototype/main.py:62-67`
- API 서버는 FastAPI 기반이며 CORS 전체 허용, SSE 스트리밍, 세션/로그/아티팩트 조회를 직접 제공한다.
  - 근거: `crewai_prototype/main.py:268-280`, `crewai_prototype/main.py:783-1245`
- 실제 연구 실행의 중심은 `ResearchCrew` 클래스다.
  - 근거: `crewai_prototype/main.py:236-242`, `crewai_prototype/main.py:988-1006`, `crewai_prototype/crew.py:57`
- `ResearchCrew`는 실행 시작 시 출력 디렉터리, 워크스페이스 scaffold, project manifest, run contract, local context store를 먼저 준비한다.
  - 근거: `crewai_prototype/crew.py:73-86`, `crewai_prototype/scaffolds/builder.py:24-61`
- 내부 역할은 6개 고정 에이전트로 구성된다.
  - `planner`, `designer`, `coder`, `executor`, `analyzer`, `writer`
  - 근거: `crewai_prototype/crew.py:170-184`
- 태스크는 factory 함수로 분리되어 있으나, 실제 오케스트레이션은 `ResearchCrew.run()` 안에 강하게 결합돼 있다.
  - 근거: `crewai_prototype/tasks/research_tasks.py:1-10`, `crewai_prototype/crew.py:2949-3513`
- 런타임 플로우는 다음 순서다.
  - 시작 로그 기록
  - 프로파일/스캐폴드/컨텍스트 준비
  - planning task
  - design task
  - iterative fix loop
  - 각 iteration마다 patch plan 생성, 코드 생성, 로컬 실행, analyzer 판단
  - 수렴 시 writing task
  - 결과 저장 및 종료 로그 기록
  - 근거: `crewai_prototype/crew.py:2951-3513`
- 코드 생성 대상은 원본 프로젝트가 아니라 매 실행마다 생성되는 run-scoped workspace다.
  - 근거: `crewai_prototype/scaffolds/builder.py:27-60`, `crewai_prototype/crew.py:102-156`
- run contract와 manifest가 존재해, 생성 코드가 어떤 엔트리포인트와 결과 산출물을 지켜야 하는지 명시한다.
  - 근거: `crewai_prototype/core/run_contract.py:18-28`, `crewai_prototype/scaffolds/builder.py:175-260`
- 기본 도구 세트는 역할별로 다르다.
  - planner/designer: Chroma
  - coder: Chroma + file write
  - executor: E2B + MLflow + file write
  - analyzer: MLflow query + Chroma
  - writer: Chroma + file write
  - 근거: `crewai_prototype/tools/__init__.py:25-47`
- 표준 로그는 JSONL이며, `research_system_ui`의 타입과 맞추려는 계약 모듈이 따로 있다.
  - 근거: `crewai_prototype/core/logger.py:4-17`, `crewai_prototype/core/api_contract.py:21-55`
- UI 연동용 API 계약은 백엔드에서 직접 노출한다.
  - 세션 목록: `/api/v1/sessions`
  - 로그 조회: `/api/v1/sessions/{run_id}/logs`
  - 로그 스트림: `/api/v1/research/{run_id}/stream`, `/api/v1/sessions/{run_id}/stream`
  - 아티팩트 조회/스트림: `/api/v1/research/{run_id}/artifacts/content`, `/api/v1/research/{run_id}/artifacts/stream`
  - 근거: `crewai_prototype/main.py:783-860`, `crewai_prototype/main.py:869-1245`
- `research_system_ui`는 실제로 이 계약을 그대로 호출한다.
  - 근거: `research_system_ui/client/src/lib/api.ts-10`, `research_system_ui/client/src/lib/api.ts-217`
- 이벤트 타입, 세션 상태, 아키텍처 타입은 UI와 백엔드가 동일한 enum 집합을 공유한다.
  - 백엔드: `crewai_prototype/core/api_contract.py:21-55`
  - 프론트: `research_system_ui/client/src/lib/types.ts-25`

#### 추론

- 현재 구조는 "에이전트 프레임워크" 자체보다 "한 클래스 안에 누적된 실행 정책"의 복잡도가 더 큰 상태다. 문제의 중심은 CrewAI보다 `ResearchCrew.run()`과 그 주변 보조 상태 파일일 가능성이 높다.
- `crewai_prototype`는 범용 멀티에이전트 플랫폼이라기보다, 실험 scaffold를 반복 수정하고 검증하는 연구 전용 오케스트레이터에 가깝다.

#### 충돌/오류 유발 후보

- `_active_runs`가 메모리 딕셔너리라 서버 재시작 시 활성 상태가 사라진다.
  - 근거: `crewai_prototype/main.py:282-284`
- 반면 세션 목록은 로그 파일에서 복구하지만, 상태 조회 `/api/v1/research/{run_id}/status`는 `_active_runs`에 없는 run_id를 404 처리한다.
  - 근거: `crewai_prototype/main.py:381-471`, `crewai_prototype/main.py:1157-1162`
- 즉 "대시보드에는 보이는데 상세 상태 엔드포인트는 실패"하는 분리 상태가 생길 수 있다.
- 태스크 팩토리 파일에 동일 이름 함수 `_patch_routing_guide`가 중복 정의되어 있다. 마지막 정의가 앞 정의를 덮어쓴다.
  - 근거: `crewai_prototype/tasks/research_tasks.py:242`, `crewai_prototype/tasks/research_tasks.py:264`, `crewai_prototype/tasks/research_tasks.py:306`
- `main.py`가 CLI, API, 세션 복구, 스트리밍, 시뮬레이션 실행까지 함께 담고 있어 변경 충돌 면적이 크다.
  - 근거: `crewai_prototype/main.py:1-1270`
- `crew.py` 단일 파일이 3246라인이며, 실제 정책과 반복 제어가 이 파일에 과밀하다.
  - 로컬 라인 수 측정값 기준

### 트랙 B. `Claude Code` 구조 파악

#### 사실

- 이 코드는 Python 연구 백엔드가 아니라 Bun/TypeScript 기반 CLI 애플리케이션이다.
  - 근거: `package.js`on:7-18`, `package.js`on:90-93`
- 패키지의 메인 및 bin 엔트리는 `cli.tsx`다.
  - 근거: `package.js`on:8-11`
- `cli.tsx`는 부팅 fast-path 라우터 역할을 한다.
  - `--version`
  - MCP 서버
  - chrome/native host
  - daemon worker
  - remote-control bridge
  - background session
  - 그 외 일반 CLI 진입
  - 근거: `cli.tsx-220`
- 본격적인 CLI와 UI 조립은 `main.tsx`에 있다.
  - 근거: `main.tsx-206`, `main.tsx`, `main.tsx`, `main.tsx`
- `main.tsx`는 Commander 기반 CLI 파서, 설정/정책/권한 초기화, 도구/명령/에이전트 로딩, REPL 실행을 모두 관장한다.
  - 근거: `main.tsx-1000`, `main.tsx-2029`
- 이 구조의 핵심 질의 엔진은 `QueryEngine` 클래스다.
  - 근거: `QueryEngine.ts-207`
- `QueryEngine`는 단일 대화 세션의 메시지, 도구 권한, 파일 캐시, 사용량, thinking 설정, MCP 클라이언트, 커맨드/툴 집합을 들고 turn 단위로 처리한다.
  - 근거: `QueryEngine.ts-183`, `QueryEngine.ts-280`
- 도구 시스템은 `Tool` 인터페이스와 `tools.ts` 레지스트리로 운영된다.
  - 근거: `Tool.ts-402`, `tools.ts-250`
- 도구는 입력/출력 스키마, permission, concurrency safety, prompt, call 메서드를 가진다.
  - 근거: `Tool.ts-402`, `Tool.ts-759`
- 실제 도구 실행 공통 경로는 `toolExecution.ts`에 있다.
  - 근거: `toolExecution.ts-131`
- slash command 시스템은 `commands.ts`가 중앙 레지스트리이며, skills/plugins/MCP/feature flag에 따라 동적으로 합성된다.
  - 근거: `commands.ts-123`, `commands.ts-259`, `commands.ts-478`
- coordinator mode가 따로 존재하며, 내부 worker를 spawn/continue/stop하는 방식의 멀티에이전트 운영을 지원한다.
  - 근거: `coordinatorMode.ts-41`, `coordinatorMode.ts-140`, `coordinatorMode.ts-237`
- task 추상화가 따로 존재하며, local shell, local agent, remote agent, in-process teammate 등 여러 작업 타입을 구분한다.
  - 근거: `Task.ts-20`, `Task.ts-76`
- bridge/remote-control 계층이 별도 디렉터리로 분리되어 있다.
  - 근거: (내부 모듈)*`, `main.tsx-4325`
- 자체 MCP 엔트리포인트도 존재한다.
  - Claude Code 본체의 MCP: `mcp.ts-196`
  - 코드 탐색용 별도 MCP 서버: `index.ts-24`, `server.ts-260`
- 별도 웹 프론트엔드가 포함되어 있다.
  - Next.js 앱: `package.js`on:5-10`
  - 웹 채팅 프록시 라우트: `route.ts-31`
  - 웹 클라이언트 기본 API URL: `http://localhost:3001`
  - 근거: `api.ts-5`

#### 추론

- `Claude Code`은 "연구 실행 백엔드"라기보다 "범용 코딩 에이전트 플랫폼 + CLI/REPL + bridge + MCP + web shell"에 가깝다.
- 멀티에이전트는 CrewAI식 고정 역할 파이프라인이 아니라, coordinator가 worker task를 동적으로 fan-out 하는 운영 모델이다.
- 따라서 이 코드를 그대로 `research_system_ui` 뒤에 붙이는 것은 "백엔드 교체"보다 "제품 아키텍처 교체"에 가깝다.

#### 구조적 위험 포인트

- `main.tsx`가 4446라인으로 매우 크다. 기능 플래그와 모드 분기가 많아 특정 모드만 떼어내 재사용하기 어렵다.
- 도구/명령/권한/설정/플러그인/스킬/MCP가 강하게 통합돼 있어, 일부만 떼어 API 서버로 노출하려면 큰 적응 계층이 필요하다.
- 웹 폴더가 있어도 그것은 현재 `research_system_ui`와 다른 계약을 사용한다.
  - 근거: `api.ts-29`, `route.ts-17`

### 트랙 C. 사실 검증 및 차이점 정리

## 3. 두 구조의 핵심 차이

### 3-1. 제품 경계

- `crewai_prototype`는 "연구 실행기"다.
  - 입력: 연구 주제, 목표, 도메인, 데이터 경로, 제약
  - 출력: run-scoped workspace, report, results, JSONL logs
- `Claude Code`는 "범용 코딩 에이전트 런타임"이다.
  - 입력: 대화, 명령, 도구 호출, bridge, MCP, worker task
  - 출력: 대화 스트림, 도구 결과, 세션 상태, 플러그인/스킬 실행

### 3-2. 에이전트 모델

- `crewai_prototype`: 6개 고정 역할 에이전트 + 고정 단계 파이프라인
  - 근거: `crewai_prototype/crew.py:170-184`, `crewai_prototype/crew.py:3046-3459`
- `Claude Code`: 기본은 단일 대화 에이전트, 필요 시 coordinator가 worker task를 동적으로 생성
  - 근거: `coordinatorMode.ts-140`, `Task.ts-20`

### 3-3. 실행 단위

- `crewai_prototype`: run_id 중심
  - 로그, 아티팩트, report, workspace가 모두 run 스코프
  - 근거: `crewai_prototype/crew.py:73-80`, `crewai_prototype/main.py:541-590`
- `Claude Code`: session/task/tool-call 중심
  - 대화 세션, background task, worker task, tool use가 핵심
  - 근거: `QueryEngine.ts-183`, `Task.ts-57`

### 3-4. UI 계약

- `research_system_ui`는 이미 `crewai_prototype`의 `/api/v1/...` 계약에 맞춰져 있다.
  - 근거: `research_system_ui/client/src/lib/api.ts-217`
- `Claude Code` 웹은 `/api/chat`, `/mcp`, health, collaboration/socket 중심이다.
  - 근거: `api.ts-29`, `api.ts-84`, `files.ts` 검색 결과
- 따라서 `Claude Code`을 대체 백엔드로 쓰려면 기존 UI를 거의 그대로 재사용하기 어렵다.

### 3-5. 기술 스택

- `crewai_prototype`: Python + CrewAI + FastAPI + JSONL
- `Claude Code`: Bun + TypeScript + React/Ink + Commander + MCP + Next.js

### 3-6. 복잡도 위치

- `crewai_prototype`의 복잡도는 연구 오케스트레이션과 실험 수렴 로직에 집중돼 있다.
- `Claude Code`의 복잡도는 플랫폼 기능 폭에 분산돼 있다.
  - CLI
  - permissions
  - tools
  - commands
  - MCP
  - bridge
  - tasks
  - web
  - plugins
  - skills

## 4. 현재 문제의 원인에 대한 구조적 해석

### 사실 기반 관찰

- 지금 UI는 `crewai_prototype` 전용 계약을 이미 사용 중이다.
- `crewai_prototype` 백엔드는 API, 세션 상태, 로그 재구성, 실험 scaffold 생성, iterative fix loop를 한 프로젝트 안에서 모두 떠안고 있다.
- `Claude Code`은 API-first 연구 백엔드가 아니라 CLI-first 범용 에이전트 플랫폼이다.

### 추론

- 지금 반복되는 "프로토콜 충돌 + 코드 오류"는 단순 버그가 아니라, 현재 시스템이 한 번에 너무 많은 책임을 묶고 있기 때문일 가능성이 높다.
- 특히 `crewai_prototype`는 UI 계약, 실험 계약, 코드 생성 계약, 실행 계약을 동시에 맞춰야 하므로 작은 수정도 넓은 회귀를 유발하기 쉽다.
- 반대로 `Claude Code`은 성숙한 에이전트 런타임 구조를 일부 참고할 가치는 있지만, 그대로 백엔드로 치환하면 연구 실행기라는 현재 목표와 제품 경계가 맞지 않는다.

## 5. 1차 결론

1. `crewai_prototype`는 이미 `research_system_ui`와 계약이 맞물린 연구 전용 백엔드다.
2. 그러나 내부는 `main.py`와 `crew.py`에 실행 책임이 과밀해 있어 충돌 면적이 크다.
3. `Claude Code`은 구조 참고 대상으로는 가치가 크지만, 현재 시스템의 "드롭인 백엔드"는 아니다.
4. 가장 현실적인 다음 단계는 `Claude Code` 전체 이식이 아니라, 그 구조에서 다음만 추출하는 것이다.
   - task/session 분리
   - tool execution 공통 경로
   - coordinator/worker 모델
   - permission/concurrency 경계
5. 즉 권장 방향은 "전면 교체"보다 "현재 API 계약을 유지한 채 오케스트레이터 재설계"다.

## 6. 다음 작업 제안

우선순위는 다음 순서를 권장한다.

1. `crewai_prototype`의 API 계약을 고정하고, UI가 의존하는 `/api/v1/...`를 별도 adapter 계층으로 분리
2. `ResearchCrew.run()`을 단계별 서비스로 분해
3. `_active_runs` 기반 메모리 상태를 영속 세션 저장소로 교체
4. 반복 루프, patch plan, local execution, analyzer 판정을 각각 독립 모듈로 절단
5. 그 이후에야 `Claude Code`식 coordinator/worker 패턴을 일부 이식 검토

## 7. 확인한 핵심 파일 목록

### `crewai_prototype`

- `crewai_prototype/main.py`
- `crewai_prototype/crew.py`
- `crewai_prototype/core/api_contract.py`
- `crewai_prototype/core/config.py`
- `crewai_prototype/core/logger.py`
- `crewai_prototype/core/run_contract.py`
- `crewai_prototype/scaffolds/builder.py`
- `crewai_prototype/tasks/research_tasks.py`
- `crewai_prototype/tools/__init__.py`

### `research_system_ui`

- `research_system_ui/client/src/lib/api.ts`
- `research_system_ui/client/src/lib/types.ts`

### `Claude Code`

- `package.js`on`
- `cli.tsx`
- `main.tsx`
- `QueryEngine.ts`
- `Tool.ts`
- `tools.ts`
- `commands.ts`
- `coordinatorMode.ts`
- `Task.ts`
- `toolExecution.ts`
- `mcp.ts`
- `index.ts`
- `server.ts`
- `package.js`on`
- `api.ts`
- `route.ts`
- (내부 모듈)architecture.md`
