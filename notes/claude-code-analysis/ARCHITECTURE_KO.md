# Claude Code 모듈 아키텍처 분석

> **출처 안내 (2026-08-20 이관)**
>
> 이 문서는 2026-04에 작성한 **본인의 아키텍처 분석 기록**입니다.
> 원래 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했고, **이 문서에 있던 원문 코드 발췌(총 35개 블록)도 함께 제거**했습니다.
>
> 남은 것은 전부 본인이 쓴 산문 분석과 직접 그린 구조도입니다.
> 공식 문서: https://docs.claude.com/en/docs/claude-code
>
> Status: finished · Written: 2026-04-08 · Sanitized: 2026-08-20

---

> 발표용 문서 — 실제 소스 파일 기반 분석 (2026-04-08)

---

## 목차

1. [전체 아키텍처 개요](#1-전체-아키텍처-개요)
2. [구역 1: 진입점·코어 엔진](#2-구역-1-진입점코어-엔진)
3. [구역 2: 툴 시스템](#3-구역-2-툴-시스템)
4. [구역 3: 서비스·MCP·멀티에이전트](#4-구역-3-서비스mcpmultiagent)
5. [구역 4: UI·상태·설정](#5-구역-4-ui상태설정)
6. [컴포넌트 간 상호작용](#6-컴포넌트-간-상호작용)
7. [핵심 설계 패턴 요약](#7-핵심-설계-패턴-요약)
8. [엔드투엔드 시나리오: 버그 수정 작업](#8-엔드투엔드-시나리오-버그-수정-작업)
5. [구역 4: UI·상태·설정](#5-구역-4-ui상태설정)
6. [컴포넌트 간 상호작용](#6-컴포넌트-간-상호작용)
7. [핵심 설계 패턴 요약](#7-핵심-설계-패턴-요약)

---

## 1. 전체 아키텍처 개요

### 계층 구조

```
┌──────────────────────────────────────────────────────────────┐
│            구역 4: UI · 상태 · 설정                           │
│  AppState(DeepImmutable) · Ink REPL · history.ts · config   │
├──────────────────────────────────────────────────────────────┤
│            구역 1: 진입점 · 코어 엔진                          │
│  main.tsx → QueryEngine → query.ts (AsyncGenerator 루프)     │
├──────────────────────────────────────────────────────────────┤
│  구역 2: 툴 시스템         │  구역 3: 서비스 · MCP · 멀티에이전트│
│  Tool<I,O> + buildTool()  │  MCP Client · Skills · AgentDef  │
│  Bash / File / Glob /     │  Coordinator Mode · Commands     │
│  Grep / Web / Agent       │  Memory · Compact Service        │
└──────────────────────────────────────────────────────────────┘
                           ↕ Claude API
```

### 실행 파이프라인 (1문장 요약)

> 사용자 입력 → `main.tsx` CLI 파싱 → `QueryEngine.submitMessage()` →
> `query.ts queryLoop()` 비동기 제너레이터 → Claude API 스트리밍 →
> 도구 호출 시 `Tool.call()` 실행 → 결과 메시지로 피드백 → 반복

---

## 2. 구역 1: 진입점·코어 엔진

### 2.1 main.tsx — 애플리케이션 진입점

**파일**: `src/main.tsx` (1,000+ 줄)

**책임**:
- 병렬 부트스트랩 초기화 (MDM 읽기, Keychain 프리페칭)
- Commander CLI 파서 등록
- 도구/명령어 레지스트리 초기화
- REPL vs. 원격 모드 분기

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

**상호작용**:
```
main.tsx ─┬─> query.ts          (쿼리 루프)
          ├─> QueryEngine.ts    (세션 상태)
          ├─> tools.ts          (도구 레지스트리)
          ├─> commands.ts       (CLI 명령어)
          └─> AppState.tsx      (전역 상태)
```

---

### 2.2 QueryEngine.ts — 세션 상태 관리자

**파일**: `src/QueryEngine.ts`

**책임**:
- 대화 메시지 히스토리 유지
- `canUseTool` 래핑 → 권한 거부 추적
- SDK 메시지 형식으로 변환

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 2.3 query.ts — 대화 루프 엔진

**파일**: `src/query.ts`

**책임**:
- Claude API 비동기 스트리밍
- 도구 호출 감지 및 실행 (`StreamingToolExecutor`)
- 토큰 예산 관리 + 자동 컴팩션
- max_output_tokens 오류 시 최대 3회 재시도

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

**데이터 흐름**:
```
사용자 메시지
  → 메시지 정규화
  → 토큰 예산 확인 (자동 컴팩션)
  → API 호출 스트리밍
  → [도구 감지?]
      YES → Tool.call() → 결과 메시지 추가 → 다음 턴
      NO  → 최종 응답 반환
```

---

### 2.4 Tool.ts — 도구 기본 타입 정의

**파일**: `src/Tool.ts` (795줄)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 2.5 tools.ts — 도구 레지스트리 팩토리

**파일**: `src/tools.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

**특징**:
- `feature()` 플래그로 도구 ON/OFF
- MCP 도구 자동 포함 (`getMcpToolsCommandsAndResources`)
- 플러그인 도구 자동 포함 (`loadAllPluginsCacheOnly`)
- `uniqBy`로 이름 충돌 해결

---

## 3. 구역 2: 툴 시스템

### 3.1 buildTool() — 안전한 도구 팩토리

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

**핵심 원칙**: 정의되지 않은 속성은 가장 제한적인 값이 기본

---

### 3.2 주요 도구별 분석

#### BashTool — 명령 실행 + 3단계 보안

```
실행 흐름:
1. 정적 AST 분석 (bashSecurity.ts, 20+ 보안 규칙)
   - ZSH 위험 명령 차단: zmodload, zpty, ztcp
   - IFS 인젝션, 명령 치환, 출력 리다이렉션 감지
2. 권한 규칙 매칭 (와일드카드)
   - "git *", "npm run:*" 패턴 자동 허용
3. ML 분류기 (선택적)
   - "allow" | "ask" | "deny" 자동 결정
4. 사용자 프롬프트 (최후 수단)
```

#### FileReadTool — 다형 파일 읽기

| 형식 | 처리 방식 |
|------|---------|
| 일반 텍스트 | 그대로 반환 (100,000자 상한) |
| 이미지 (PNG/JPG) | Base64 + 자동 축소 |
| PDF | 페이지 범위 선택 추출 |
| Jupyter (.ipynb) | 셀·출력 구조화 반환 |

보안: `/dev/zero`, `/dev/urandom` 차단

#### FileEditTool — 정확한 문자열 교체

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### GlobTool / GrepTool — 검색 도구

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 3.3 AgentTool — 서브에이전트 스폰

**파일**: `src/tools/AgentTool/`

#### 도구 필터링 메커니즘

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### Fork 서브에이전트 (암시적 포크)

**파일**: `src/tools/AgentTool/forkSubagent.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### Worktree 격리

**파일**: `src/tools/EnterWorktreeTool/`

```
EnterWorktreeTool 실행 순서:
1. Git worktree add <path> -b <branch>
2. node_modules 등 무거운 디렉토리 심볼릭 링크
3. .gitignore, Git 설정 상속
4. CWD 전환 → 메모리 캐시 초기화 (CLAUDE.md 재로드)

ExitWorktreeTool:
1. 미커밋 변경 확인 → 사용자에게 알림
2. Tmux 세션 종료 (선택)
3. git worktree remove
4. 원래 CWD 복귀
```

---

### 3.4 권한 처리 시스템

```
PermissionMode 계층:
  default → ask → allow → deny → bypass → plan

Tool.checkPermissions() 3단계:
  1. 정적 분석 (BashTool AST)
  2. 와일드카드 규칙 매칭 ("git *", "npm run:*")
  3. ML 분류기 → 최종 사용자 프롬프트

PermissionResult:
  { behavior: 'allow' | 'deny' | 'prompt' | 'bypass' }
```

---

## 4. 구역 3: 서비스·MCP·멀티에이전트

### 4.1 MCP (Model Context Protocol) 클라이언트

**파일**: `src/mcp/client.ts` (3,349줄), `config.ts` (51KB), `auth.ts` (88KB)

**책임**:
- 다중 MCP 서버 연결 상태 관리
- OAuth 인증 및 토큰 캐싱
- 도구/리소스/프롬프트 로드
- 자동 재연결 (exponential backoff)

#### 서버 연결 타입

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### MCP 연결 라이프사이클

```
1. 설정 수집: .mcp.json + settings.json + plugins
   └─> dedupPluginMcpServers() + dedupClaudeAiMcpServers()

2. 연결 시작: useManageMCPConnections.ts
   └─> ensureConnectedClient(name)
       ├─ Stdio: 로컬 프로세스 스폰
       ├─ HTTP/SSE: fetch + OAuth 토큰 주입
       └─ WebSocket: ws:// 연결

3. 서버 초기화:
   ├─ ListTools → MCP 도구 목록
   ├─ ListPrompts → MCP 스킬 발견
   └─ ListResources → 리소스

4. 상태: Connected | NeedsAuth | Failed
   └─ 실패 시 최대 5회 재연결 (exponential backoff)
```

#### 도구명 정규화

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 파일 쓰기 원자성

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 4.2 스킬 & 커맨드 시스템

**파일**: `src/skills/`, `src/commands.ts`

#### 스킬 로딩 소스

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 스킬 프론트매터

```yaml
---
description: 스킬의 한 줄 설명
when-to-use: 모델이 언제 써야 할지 (모델용 휴리스틱)
allowed-tools: [bash, file-read, file-edit]
model: claude-3-5-sonnet  # 또는 'inherit'
disableModelInvocation: false
user-invocable: true
---
스킬 마크다운 본문...
```

#### 모델 호출 가능 조건

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 스킬 실행 라우팅 (SkillTool)

```
모델 → SkillTool("review-code", args)
  ├─ loadedFrom === 'mcp'? → MCP 서버에 call_tool
  ├─ context === 'fork'?   → Fork 서브에이전트
  └─ else?                  → 프롬프트 렌더링 + 로컬 실행
```

---

### 4.3 멀티에이전트 시스템

**파일**: `src/tools/AgentTool/loadAgentsDir.ts`, `agentMemory.ts`

#### 에이전트 정의

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 에이전트 메모리 (agentMemory.ts)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 에이전트 소스 우선순위

```
우선순위 (높을수록 앞):
  policySettings > plugin > userSettings > projectSettings > built-in

소스가 높은 것이 낮은 것을 "override" 표시
```

---

### 4.4 코디네이터 모드

**파일**: `src/coordinator/coordinatorMode.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### Task Notification XML (Worker → Coordinator 완료 알림)

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed</status>  <!-- completed | failed | killed -->
  <summary>Agent "Investigate auth bug" completed</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  <usage>
    <total_tokens>12345</total_tokens>
    <tool_uses>8</tool_uses>
    <duration_ms>45000</duration_ms>
  </usage>
</task-notification>
```

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 4.5 Bridge (IDE 통신)

**파일**: `src/bridge/`

**목적**: VSCode 등 IDE와 Claude Code CLI 간 양방향 통신 레이어

#### Transport 레이어

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 메시지 흐름

```
IDE → Claude:
  SDKMessage 수신 → 중복 감지 (BoundedUUIDSet 100개) → handleIngressMessage()
  ├─ isSDKControlRequest → onControlRequest (권한/모델 설정 요청)
  └─ isSDKMessage → onInboundMessage (일반 메시지)

Claude → IDE:
  Message → makeResultMessage() → SDKMessage 변환 → bridgeHandle.writeMessages()

Bridge 안전 명령 (모바일에서 사용 가능):
  compact, clear, cost, summary, releaseNotes, files
  + 모든 PromptCommand (스킬) → type: 'local-jsx'만 불가
```

---

### 4.6 메모리 시스템 (CLAUDE.md / memdir)

**파일**: `src/memdir/`

#### 메모리 디렉터리 구조

```
~/.claude/projects/{project-slug}/memory/
├── MEMORY.md              # 인덱스 (최대 200줄 / 25KB)
├── decisions/auth.md
├── architectures/system-design.md
└── learnings/common-patterns.md
```

#### MEMORY.md 로딩 제한

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### 시스템 프롬프트 주입

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 4.7 컨텍스트 압축 (Compact Service)

**파일**: `src/services/compact/` (compact.ts 60KB, autoCompact.ts, microCompact.ts)

**압축 목표**: 오래된 메시지 요약 → 토큰 절약

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 4.8 주요 MCP 고급 기능

| 기능 | 파일 | 역할 |
|------|------|------|
| OAuth 인증 | `auth.ts` (88KB) | 토큰 획득, 갱신, 캐싱 |
| 엘리시테이션 | `elicitationHandler.ts` | MCP → 사용자 폼/URL 입력 |
| 채널 알림 | `channelNotification.ts` | Slack/Discord 스타일 채널 |
| 환경 변수 확장 | `envExpansion.ts` | `${API_KEY}` → 실제 값 |
| 리소스 제한 | `mcpValidation.ts` | 설명 2048자 제한, 내용 잘라내기 |

---

## 5. 구역 4: UI·상태·설정

### 5.1 AppState — 전역 상태 저장소

**파일**: `src/state/AppStateStore.ts`, `src/state/AppState.tsx`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### Store.ts — 제네릭 상태 저장소

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### onChangeAppState.ts — 변경 부수 효과

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 5.2 상태 관리 패턴

#### Provider 계층 구조

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

#### React 훅 목록

| 훅 | 역할 |
|---|------|
| `useAppState()` | 전체 AppState 읽기 (useSyncExternalStore) |
| `useSetAppState()` | AppState 업데이트 |
| `useNotifications()` | 알림 추가/제거 |
| `useSettings()` | 현재 설정 읽기 |
| `useArrowKeyHistory()` | Up/Down 화살표 히스토리 탐색 |
| `useHistorySearch()` | Ctrl+R 히스토리 검색 |
| `useReplBridge()` | Always-on Bridge 통신 |

---

### 5.3 KeybindingContext — 키 입력 처리

**파일**: `src/keybindings/KeybindingContext.tsx`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 5.4 history.ts — 명령어 히스토리

**파일**: `src/history.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 5.5 cost-tracker.ts — 비용 추적

**파일**: `src/cost-tracker.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

### 5.6 API 클라이언트 — 멀티 프로바이더

**파일**: `src/api-client.ts` (또는 `services/api/`)

```
지원 프로바이더:
  Direct        → api.anthropic.com
  AWS Bedrock   → AWS 자격증명 기반
  GCP Vertex    → GCP 자격증명 기반
  Azure         → Azure AI 엔드포인트

공통 인터페이스:
  createMessage() → BetaMessageStreamParams
  스트리밍 응답 → AsyncGenerator<StreamEvent>
```

---

### 5.7 bootstrap/state.ts — 서버 측 싱글톤

**파일**: `src/bootstrap/state.ts`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## 6. 컴포넌트 간 상호작용

### 6.1 도구 호출 전체 흐름

```
사용자: "이 파일 수정해줘"
  ↓
main.tsx → submitMessage()
  ↓
QueryEngine: 권한 추적 래퍼 등록
  ↓
query.ts queryLoop: Claude API 호출
  ↓
API 응답: tool_use { name: "FileEdit", input: {...} }
  ↓
findToolByName(tools, "FileEdit") → FileEditTool
  ↓
FileEditTool.checkPermissions() → PermissionResult
  ├─ allow: 실행
  └─ prompt: 사용자 확인 후 실행
  ↓
FileEditTool.call(args, context)
  ├─ 파일 읽기 → 문자열 매칭
  ├─ 수정 적용
  └─ Git diff 생성
  ↓
ToolResult → 다음 메시지로 추가 → 다음 턴
```

### 6.2 MCP 도구 호출 흐름

```
모델 → "mcp__Slack__send_message" 도구 호출
  ↓
MCPTool (정규화된 이름으로 라우팅)
  ↓
ensureConnectedClient("Slack")
  ├─ 캐시 히트? → 기존 클라이언트 재사용
  └─ 미스? → 새 연결 생성 (OAuth 포함)
  ↓
MCP 서버에 call_tool 요청 (JSON-RPC)
  ↓
결과 반환 → 모델
```

### 6.3 서브에이전트 스폰 흐름

```
모델 → AgentTool({ subagent_type: "explore", prompt: "..." })
  ↓
AgentTool.call()
  ├─ 에이전트 정의 로드 (loadAgentsDir)
  ├─ 도구 풀 필터링 (filterToolsForAgent)
  ├─ MCP 서버 초기화 (에이전트 전용)
  └─ 에이전트 메모리 로드 (agentMemory.ts)
  ↓
새 QueryEngine 생성 (자식 세션)
  ├─ 부모로부터 메시지 히스토리 상속 (fork 시)
  └─ 에이전트별 시스템 프롬프트 적용
  ↓
자식 세션 실행 → 결과 부모에게 반환
  ↓
MCP 서버 정리 (에이전트 전용 것만)
```

---

## 7. 핵심 설계 패턴 요약

### 7.1 비동기 제너레이터 (Streaming)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 7.2 Fail-Closed 보안

```
기본값 = 가장 제한적
  buildTool() → isConcurrencySafe: false, isReadOnly: false
  MCP 연결 실패 → FailedMCPServer (도구 비활성)
  권한 규칙 없음 → 사용자에게 물어봄
```

### 7.3 메모이제이션 + 캐시 무효화

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 7.4 Feature Flag 기반 점진적 활성화

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 7.5 의존성 주입

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 7.6 원자적 파일 쓰기

```
임시 파일 → 데이터 쓰기 → datasync() → rename()
→ 중간 실패 시 원본 손상 없음
```

---

## 부록: 주요 파일 맵

| 구역 | 파일 | 역할 |
|------|------|------|
| 코어 | `src/main.tsx` | CLI 진입점 |
| 코어 | `src/QueryEngine.ts` | 세션 상태 관리 |
| 코어 | `src/query.ts` | 대화 루프 엔진 |
| 코어 | `src/Tool.ts` | 도구 기본 타입 |
| 코어 | `src/tools.ts` | 도구 레지스트리 |
| 툴 | `src/tools/BashTool/` | 명령 실행 |
| 툴 | `src/tools/FileReadTool/` | 파일 읽기 |
| 툴 | `src/tools/FileEditTool/` | 파일 편집 |
| 툴 | `src/tools/AgentTool/` | 서브에이전트 |
| 툴 | `src/tools/EnterWorktreeTool/` | 격리 환경 |
| MCP | `src/mcp/client.ts` | MCP 클라이언트 (3,349줄) |
| MCP | `src/mcp/config.ts` | MCP 설정 (51KB) |
| MCP | `src/mcp/auth.ts` | OAuth (88KB) |
| 스킬 | `src/skills/loadSkillsDir.ts` | 스킬 발견 |
| 에이전트 | `src/tools/AgentTool/loadAgentsDir.ts` | 에이전트 정의 |
| 에이전트 | `src/tools/AgentTool/forkSubagent.ts` | Fork 구현 |
| 에이전트 | `src/tools/AgentTool/agentMemory.ts` | 에이전트 메모리 |
| 상태 | `src/state/AppStateStore.ts` | AppState 타입 |
| 상태 | `src/state/store.ts` | 제네릭 스토어 |
| 상태 | `src/state/onChangeAppState.ts` | 변경 부수 효과 |
| UI | `src/history.ts` | 명령어 히스토리 JSONL |
| UI | `src/cost-tracker.ts` | 비용 추적 |
| UI | `src/keybindings/KeybindingContext.tsx` | 키바인딩 |
| 설정 | `src/utils/config.ts` | Project/Global 설정 |
| 설정 | `src/bootstrap/state.ts` | 서버 싱글톤 |

---

## 8. 엔드투엔드 시나리오

### 시나리오 목록

| 문서 | 사용자 입력 | 핵심 도구 |
|------|------------|---------|
| [SCENARIO_KO.md](SCENARIO_KO.md) | "CSV 합쳐서 엑셀로 저장해줘" | Write + BashTool |
| [SCENARIO_GITHUB_KO.md](SCENARIO_GITHUB_KO.md) | "GitHub 이슈 우선순위별로 정리해줘" | **MCPTool** |
| [SCENARIO_WEBFETCH_KO.md](SCENARIO_WEBFETCH_KO.md) | "React Query vs SWR 조사해서 위키에 저장해줘" | **WebFetchTool** |
| [SCENARIO_PROFILING_KO.md](SCENARIO_PROFILING_KO.md) | "API 병목 어디인지 찾아줘" | **BashTool(진단)** |

### 시나리오별 도구 패턴 비교

```
CSV → 엑셀:     Glob → Read → Bash(확인) → Write → Bash(실행)
                FileEditTool 0회 / BashTool이 실제 작업 수행

GitHub 이슈:    Bash(git remote) → MCPTool × 3 → Write
                MCP JSON-RPC → GitHub REST API 흐름 최초 등장

웹 조사:        WebFetch × 3 → Write
                HTML→Markdown 변환 + 도메인별 권한 확인 패턴

성능 분석:      Bash × 2(측정) → Grep → Read → Write
                BashTool이 실행이 아닌 진단 도구로 사용
                결과를 Claude가 추론 체인으로 이어가는 패턴
```

---

### 데이터 정리 후 엑셀 저장 요약

> 상세 문서: [SCENARIO_KO.md](SCENARIO_KO.md)

**사용자 입력**: `"sales 폴더에 이번 달 CSV 파일들 있는데, 상품별로 매출 합산해서 엑셀로 저장해줘"`

코드 수정이 없는 **결과물 생성** 작업. FileEditTool 대신 Write + BashTool이 핵심이 된다.

### 통과 순서

```
main.tsx → QueryEngine → queryLoop
  ├─ 턴1: GlobTool        → [결과①] sales/*.csv 5개 경로
  ├─ 턴2: FileReadTool    → [결과②] CSV 컬럼 구조 확인
  ├─ 턴3: BashTool        → [y] pandas/openpyxl 설치 확인
  ├─ 턴4: Write           → [y] summarize.py 스크립트 생성
  ├─ 턴5: BashTool        → [y] python3 실행 → xlsx 파일 생성
  ├─ 턴6: BashTool        → 결과 파일 검증 (상위 5개 확인)
  └─ 턴7: 최종 텍스트 응답
```

### 핵심 포인트

| 포인트 | 내용 |
|--------|------|
| **FileEditTool 0회** | 기존 코드를 건드리지 않음 |
| **Write 도구** | 처리 스크립트를 처음부터 새로 작성 |
| **BashTool이 실제 작업** | 검증용이 아닌 실제 데이터 처리 수행 |
| **사용자 확인 3회** | Bash 실행마다 매번 확인 (실행 내용이 달라서) |
| **결과물** | 코드 변경이 아닌 `.xlsx` 파일 생성 |
| **총 비용** | ~$0.0009 |

### 타임라인 요약

```
T+0.0s  입력 수신
T+0.2s  GlobTool → CSV 5개 발견
T+0.4s  FileReadTool → 컬럼 구조 파악
T+0.6s  BashTool → [y] 라이브러리 확인 (340ms)
T+4.0s  Write → [y] summarize.py 생성
T+7.6s  BashTool → [y] python3 실행 (1.8s)
T+9.5s  BashTool → 결과 검증 (자동)
T+10.2s 최종 응답 출력 완료
──────────────────────────────────────────
총 소요: ~10.2초 / API 7턴 / 도구 6회
생성 파일: summarize.py + 매출요약_202604.xlsx
```

### Write vs FileEditTool

```
FileEditTool  → 있는 파일의 특정 부분을 old→new 교체
Write         → 없는 파일을 처음부터 전체 내용으로 생성

데이터 처리 작업은 수정할 코드가 없기 때문에
Write로 스크립트를 만들고 BashTool로 실행하는 패턴을 택한다.
```

