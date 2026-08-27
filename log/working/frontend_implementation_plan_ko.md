# 프론트엔드 구현 계획서 — CrewAI 파이프라인 안정화 연동

**참여 전문가**
- **Alex** — Codex 설계 엔지니어 (파이프라인 안정성 / 백엔드 아키텍처)
- **Sam** — Claude Code 설계 엔지니어 (에이전트 워크플로우 / 장기 실행 태스크)
- **Jordan** — 시니어 프론트엔드 엔지니어 10년차 (React / 실시간 UI / AI-native UX)

---

## 1. 도입 — Jordan의 현황 진단

**Jordan:** 두 분이 지난 Sprint에서 백엔드를 꽤 크게 바꿨더군요. 제가 코드베이스를 돌아보니 당장 손봐야 할 부분이 몇 가지 보입니다.

현재 `research_system_ui`는 React 19, Tailwind v4, Framer Motion 12, Radix UI 조합이라 기술 스택 자체는 훌륭해요. 그런데 구현 방식이 일부 2022년 패턴에 머물러 있습니다.

크게 두 가지 차원에서 문제가 있습니다.

첫째, **백엔드가 새로 만든 이벤트들이 UI에 아무런 처리기가 없습니다.** `exec_stdout`, `PREFLIGHT_QUESTION`, `failure_escalation`, `extension_proposals`, `token_budget_warning` — 이 다섯 가지 이벤트가 지금 프론트엔드에 도달해도 그냥 무시되거나 `AGENT_MESSAGE`와 똑같이 밋밋하게 출력됩니다.

둘째, **Gate 컴포넌트들이 UX를 막아버립니다.** `ApprovalDialog`는 전체 화면을 덮는 모달이고, `GuidancePanel`은 화면 상단 배너 형태입니다. 실험이 몇 시간씩 돌아가는 동안 사용자가 잠깐 자리를 비웠다 돌아왔을 때, 무언가를 요청받고 있다는 사실을 즉시 파악하기 어렵습니다.

**Alex:** 그 외에도 이번에 `CheckpointManager`가 생겼으니, 파이프라인이 중간에 죽었다가 재시작하는 경우 UI에서도 "이 실행은 Phase 2에서 재개됐습니다" 같은 맥락이 보여야 합니다.

**Sam:** `PreflightClarifier`도 핵심입니다. 실행 시작 직후 최대 4개 질문이 60초 타임아웃으로 순서대로 옵니다. 사용자가 놓치면 기본값으로 진행되니, "이 질문에 지금 답해야 합니다"를 명확히 전달해야 해요. 모달로 막아버리면 너무 강압적이고, 배너로 처리하면 놓치기 쉽습니다.

**Jordan:** 정확해요. 그래서 저는 크게 세 가지 방향으로 접근하려 합니다.

1. **새 이벤트 타입 전부 처리** — `types.ts` 확장 + `LogEvents.tsx` 렌더러 추가
2. **Gate UX 개선** — 모달/배너 → Drawer + Ambient 알림 패턴
3. **신규 컴포넌트 5개** — `PreflightFlow`, `TerminalPane`, `TokenBudgetBar`, `ProposalSheet`, `RunStatusRibbon`

---

## 2. 현재 프론트엔드 구조 분석

### 2-A. 강점

**Jordan:**

```
components/
  ApprovalDialog.tsx    # Radix Dialog — 구조는 좋음, UX만 개선 필요
  GuidancePanel.tsx     # 배너 형태 — Drawer로 교체 대상
  LogView.tsx           # Phase stepper + 이벤트 목록 — 단단한 기반
  LogEvents.tsx         # 이벤트 타입 → 컴포넌트 매핑 — 확장 필요
  DetailPanel.tsx       # 메타데이터 인스펙터 — 잘 동작
```

`api.ts`의 SSE 스트리밍 레이어는 깔끔합니다. EventSource를 직접 다루는 부분이 `useResearch` 훅 안에 잘 캡슐화돼 있고, 중복 이벤트 필터링도 있어요. 이 부분은 손댈 게 없습니다.

### 2-B. 개선이 필요한 부분

**Jordan:**

**① 이벤트 타입 매핑이 수동입니다.**

`LogEvents.tsx`를 보면 이렇습니다:

```tsx
// 현재 — switch/if 체인으로 수동 매핑
if (event.event_type === 'AGENT_MESSAGE') return <AgentMessage ... />
if (event.event_type === 'SYSTEM_START') return <SystemBanner ... />
// Sprint 1-4 신규 이벤트들은 처리기 없음
```

신규 이벤트가 생길 때마다 이 파일을 열어서 분기를 추가해야 합니다. 레지스트리 패턴으로 바꿔야 확장성이 생깁니다.

**② Phase 감지가 문자열 파싱에 의존합니다.**

```tsx
// LogView.tsx — 현재
function detectPhases(events: LogEvent[]) {
  return events.filter(e => e.content.includes('[Phase'));
  // "[Phase 2]" 같은 문자열을 content에서 긁어냄 — brittle
}
```

이제 백엔드가 `metadata: { phase: 2 }` 구조로 정보를 내려주니, content 파싱 대신 metadata를 읽어야 합니다.

**③ GuidancePanel이 배너로 구현돼 있어 파악이 어렵습니다.**

```tsx
// SessionView.tsx — 현재
{guidancePayload && (
  <div className="fixed top-0 left-0 right-0 z-50 ...">
    <GuidancePanel payload={guidancePayload} ... />
  </div>
)}
```

상단 고정 배너는 스크롤해서 지나치면 보이지 않습니다.

**Alex:** 백엔드 입장에서 추가하면, 이제 `GuidanceGate`가 Phase 2 repair loop 뿐만 아니라 `PreflightClarifier`와 `ExitGate` 실패 시에도 발동됩니다. UI 진입점이 세 곳으로 늘어났습니다.

**Sam:** `ContextInjectionQueue`도 있어요. 실행 중에 사용자가 추가 컨텍스트를 던져넣을 수 있어야 합니다. 지금 UI에는 그 입력창이 없습니다.

---

## 3. 새 이벤트 타입과 UI 요구사항 매핑

**Jordan:** 새 이벤트를 하나씩 짚고 어떤 컴포넌트가 필요한지 정리하겠습니다.

| 이벤트 타입 | 발생 시점 | 필요한 UI 처리 |
|---|---|---|
| `PREFLIGHT_QUESTION` | Phase 0 직후, 최대 4회 | **PreflightFlow** — 타임다운 카드 시퀀스 |
| `PREFLIGHT_ANSWERED` | 각 질문 응답 후 | 카드 완료 표시 |
| `exec_stdout` | Phase 3 실험 중 | **TerminalPane** — 실시간 스트리밍 뷰 |
| `token_budget_warning` | Phase 2 진행 중 | **TokenBudgetBar** — 경고 배지 |
| `token_budget_snapshot` | Phase 2 진행 중 | TokenBudgetBar 업데이트 |
| `failure_escalation` | Phase 3 반복 실패 시 | **FailureEscalationAlert** — 붉은 배너 |
| `extension_proposals` | Phase 4 완료 후 | **ProposalSheet** — 추가 실험 카드 |
| `checkpoint_saved` (향후) | 각 Phase 완료 시 | **RunStatusRibbon** — 재개 가능 표시 |

**Sam:** `PREFLIGHT_QUESTION`은 타임아웃이 60초입니다. UI에서 카운트다운을 보여줘야 사용자가 급박함을 느끼고 반응하죠. 아니면 60초 후 "기본값으로 진행됩니다"라는 피드백이라도.

**Jordan:** 맞아요. `useCountdown` 훅을 만들어서 각 카드에 progress bar로 시각화합니다. 타임아웃이 지나면 해당 카드를 "기본값 사용됨"으로 dimmed 처리하고 다음 질문으로 넘어가요.

**Alex:** `exec_stdout` 이벤트는 실험 스크립트가 실행되는 내내 초당 수십 개씩 올 수 있습니다. 가상 스크롤링 없이 DOM에 그대로 렌더링하면 메모리 문제가 생깁니다.

**Jordan:** 이미 Recharts가 들어가 있으니 성능 선례는 있는데, 터미널 뷰는 별개입니다. `@tanstack/react-virtual` 또는 직접 구현한 windowed list로 처리할 생각입니다. 최근 3000줄만 DOM에 유지하고, 스크롤이 맨 아래 있으면 자동 스크롤, 사용자가 위로 올리면 고정(auto-scroll lock)하는 패턴이 표준입니다.

---

## 4. 기존 컴포넌트 개선 계획

### 4-A. GuidancePanel → Drawer 패턴 전환

**Jordan:**

현재 배너 방식의 문제:
- 스크롤 시 화면 밖으로 나가 놓칠 수 있음
- 여러 파일에 대해 동시에 guidance가 필요할 경우 스택 처리 불가
- 배너 아래 콘텐츠가 가려짐

대안: **오른쪽 Drawer** (Radix UI `Sheet` 컴포넌트 활용)

```tsx
// 변경 후
<Sheet open={!!guidancePayload} onOpenChange={...}>
  <SheetContent side="right" className="w-[480px]">
    <GuidanceDrawer payload={guidancePayload} onSubmit={handleGuidance} />
  </SheetContent>
</Sheet>
```

- 로그 뷰가 Drawer 뒤에 계속 보임 (컨텍스트 유지)
- 최소화 버튼으로 잠시 접어뒀다가 다시 펼치기 가능
- 퍼블릭 알림 배지 (우하단 고정) — 실험이 입력을 기다리고 있음을 상시 표시

**Sam:** 좋습니다. 그리고 `guidance_status` API가 있으니 세션에 재진입할 때도 대기 중인 gate를 복원할 수 있습니다.

**Jordan:** 맞아요. `SessionView` 마운트 시 `GET /api/v1/runs/{run_id}/guidance_status`를 호출해서 대기 중 guidance가 있으면 Drawer를 자동으로 열도록 합니다.

### 4-B. ApprovalDialog — 레이아웃 개선

**Jordan:**

전체화면 모달은 유지하되 내용 구조를 개선합니다:

- 현재: 탭 2개 (계획 / 파일)
- 변경: 3열 레이아웃 (계획 요약 | 파일 트리 | 수정 피드백)
- 큰 화면에서 전체 정보를 한눈에 파악 가능

`plan_bundle`의 `planner.recommended_profile`을 뱃지로 표시하고, `designer.entry_point`와 파일 수를 헤더에 요약합니다.

### 4-C. LogEvents.tsx — 레지스트리 패턴

**Jordan:**

```tsx
// 변경 후: 등록 기반 렌더러
const EVENT_RENDERERS: Record<string, React.FC<{ event: LogEvent }>> = {
  SYSTEM_START:          SystemBanner,
  SYSTEM_END:            SystemBanner,
  AGENT_MESSAGE:         AgentMessage,
  PHASE_START:           PhaseMarker,
  PHASE_COMPLETE:        PhaseMarker,
  USER_GUIDANCE_NEEDED:  GuidanceTrigger,
  PREFLIGHT_QUESTION:    PreflightCard,
  PREFLIGHT_ANSWERED:    PreflightAnswered,
  exec_stdout:           StdoutLine,
  token_budget_warning:  TokenBudgetWarning,
  token_budget_snapshot: null,           // 이벤트 목록에 표시하지 않음 (sidebar로만)
  failure_escalation:    FailureAlert,
  extension_proposals:   ProposalTrigger,
  SECTION_DRAFT_DONE:    SectionDone,
};

// 렌더링
export function EventRenderer({ event }: { event: LogEvent }) {
  const Renderer = EVENT_RENDERERS[event.event_type] ?? DefaultEvent;
  if (Renderer === null) return null;  // 숨김 처리
  return <Renderer event={event} />;
}
```

새 이벤트 타입을 추가할 때 이 파일만 열면 됩니다.

### 4-D. LogView.tsx — Phase stepper 메타데이터 기반으로 전환

**Jordan:**

```tsx
// 변경 전
const phases = events.filter(e => e.content.includes('[Phase'));

// 변경 후
const phases = events.filter(
  e => ['PHASE_START', 'PHASE_COMPLETE'].includes(e.event_type) && e.metadata?.phase != null
);
const currentPhase = phases.findLast(e => e.event_type === 'PHASE_START')?.metadata?.phase ?? 0;
```

Phase stepper에 체크포인트 재개 표시도 추가:
```tsx
{isResumed && (
  <Tooltip content={`Phase ${resumedFromPhase}에서 재개됨`}>
    <RotateCcw className="h-3 w-3 text-amber-400" />
  </Tooltip>
)}
```

---

## 5. 신규 컴포넌트 설계

### 5-A. PreflightFlow

**Jordan:**

```
┌─────────────────────────────────────────────────────────────┐
│  실행 전 확인 (4개 중 2번째)                    ⏱ 00:47     │
├─────────────────────────────────────────────────────────────┤
│  GPU 메모리나 CPU 코어 수 등 컴퓨팅 제약이 있나요?          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 기본값: No specific constraints — use available ... │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  [내 답변 입력...                                         ] │
│                                                             │
│  [기본값으로 진행]              [이 답변으로 계속하기  →]   │
│  ████████████████████░░░░░░░░░░░░░  타임아웃 47초 남음      │
└─────────────────────────────────────────────────────────────┘
```

- 화면 중앙 고정 카드 (모달 아님 — 뒤에 이벤트 로그가 흐릿하게 보임)
- `PREFLIGHT_QUESTION` 이벤트 수신 → 카드 표시
- `PREFLIGHT_ANSWERED` 수신 → 카드 완료 상태로 전환 후 다음 질문으로
- 타임아웃 시 자동으로 "기본값 사용" 표시 후 다음 질문 대기

**API 연동:**
```
POST /api/v1/runs/{run_id}/guidance
  { file_path: "preflight_{question_key}", user_action: "provide_fix", hint: "사용자 답변" }
```

기존 guidance 엔드포인트를 재사용합니다 (`PreflightClarifier`가 `GuidanceGate` 패턴을 씁니다).

**Sam:** 기존 `/guidance` 엔드포인트를 재사용하는 게 맞습니다. 백엔드가 이미 `preflight_{key}`를 gate key로 쓰고 있으니 프론트엔드에서도 별도 엔드포인트 불필요합니다.

**Jordan:** 좋아요. 그럼 `file_path`가 `"preflight_"` 로 시작하는 guidance 이벤트는 `PreflightCard`로, 아닌 건 기존 `GuidanceDrawer`로 분기하면 됩니다.

```tsx
function routeGuidanceEvent(payload: GuidancePayload) {
  if (payload.entry.startsWith('preflight_')) {
    setPreflightPayload(payload);
  } else {
    setGuidancePayload(payload);
  }
}
```

### 5-B. TerminalPane

**Jordan:**

```
┌─────────────────────────────────────────────────────────────┐
│  실험 실행 중  python src/main.py         [▼ 자동 스크롤]  │
├─────────────────────────────────────────────────────────────┤
│  $ python src/main.py                                       │
│  Loading dataset: CIFAR-10...                               │
│  Epoch 1/50: loss=2.3041, acc=0.1023                       │
│  Epoch 2/50: loss=2.1203, acc=0.2156                       │
│  ▋                                              (라이브)    │
└─────────────────────────────────────────────────────────────┘
```

구현 포인트:
- `exec_stdout` 이벤트 → `stdout_lines` state 배열에 `append`
- `@tanstack/react-virtual` 로 가상화 (최대 5000줄 유지, 초과 시 앞부분 제거)
- `autoScroll` state: 스크롤이 맨 아래 → true, 사용자가 위로 올리면 false (자동 잠금)
- `[▼ 자동 스크롤]` 버튼으로 수동 토글
- 배경: `bg-zinc-950`, 폰트: `font-mono text-xs`, 텍스트: `text-green-400`
- Phase 3 완료 시 "실험 완료 — Return code: 0" 마지막 줄 추가

**Alex:** 이 뷰는 Phase 3에서만 보여야 합니다. Phase 2 repair 중에 나오는 stdout이랑 섞이면 혼란스럽습니다.

**Jordan:** 동의합니다. `currentPhase === 3` 일 때만 TerminalPane을 LogView 하단에 append하겠습니다. 그리고 Tab으로 "이벤트 로그 / 터미널" 전환 가능하게 합니다.

### 5-C. TokenBudgetBar

**Jordan:**

```
Phase 2 코딩 중                      토큰 예산 ████████░░ 82%  ⚠
```

- Phase 2 헤더 아래 얇은 Progress bar (Radix UI `Progress` 컴포넌트)
- `token_budget_snapshot` → 값 업데이트 (애니메이션 전환)
- `token_budget_warning` (80% 이상) → 주황색, 경고 아이콘 표시
- 100% 도달 → 빨간색 ("의존성 컨텍스트 한계 도달. LLM 입력이 절단되고 있습니다.")
- Phase 2 완료 시 bar 제거

**Sam:** 사용자가 이게 뭔지 모를 수 있으니 툴팁이 필요합니다. "LLM이 파일 간 의존성을 학습하는 데 사용한 컨텍스트 크기" 정도면 충분합니다.

**Jordan:** 네, Radix `Tooltip`으로 처리합니다.

### 5-D. ProposalSheet

**Jordan:**

```
┌─────────────────────────────────────────────────────────────┐
│  추가 실험 제안 (3개)                              [닫기 ×] │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────┐                         │
│  │ 🔁 다른 시드로 재현성 확인     │  [이 실험 실행하기]     │
│  └────────────────────────────────┘                         │
│  ┌────────────────────────────────┐                         │
│  │ ✂️ Ablation study 실행         │  [이 실험 실행하기]     │
│  └────────────────────────────────┘                         │
│  ┌────────────────────────────────┐                         │
│  │ 🌍 다른 데이터셋으로 일반화 검증│  [이 실험 실행하기]     │
│  └────────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

- `extension_proposals` 이벤트 → Bottom sheet (Drawer, 아래에서 올라오는 형태)
- 각 제안에 "이 실험 실행하기" 버튼 → 기존 연구 주제 + 제안을 합쳐서 새 run 생성
- "나중에" 버튼으로 sheet 닫기 가능 (이후 결과 탭에서 다시 확인 가능)
- 실험 실패 시 제안 내용이 다르므로 (실패 원인 조사 제안) 실패/성공 뱃지 구분

**새 API 엔드포인트 필요:**
```
POST /api/v1/runs/{run_id}/extension/accept
  { proposal_index: 0, proposal_text: "..." }
→ 새 run 생성 후 run_id 반환
```

**Alex:** 이 엔드포인트는 백엔드에 아직 없습니다. 현재는 `POST /api/v1/research`로 새 topic을 조합해 새 run을 만드는 방식으로 임시 구현하면 됩니다.

**Jordan:** 그렇게 처리하겠습니다. proposal_text를 goal 필드에 이어붙여서 `POST /api/v1/research`를 호출하면 됩니다. 나중에 엔드포인트가 생기면 교체만 하면 되죠.

### 5-E. RunStatusRibbon

**Jordan:**

```
┌─────────────────────────────────────────────────────────────┐
│  run_id: abc123def456      Phase 2 코딩 중   ↺ 재개됨 (P2) │
│  경과: 01:24:33             파일 7/12 완료    ⚡ 35 repairs │
└─────────────────────────────────────────────────────────────┘
```

- SessionView 상단 고정 리본 (항상 표시)
- 실행 경과 시간 (`useElapsedTime` 훅, 1초마다 업데이트)
- Phase/파일 진행 상황
- 체크포인트 재개 여부 (`AGENT_MESSAGE` 중 "Resuming from checkpoint" 감지)
- repair 누적 횟수 (`USER_GUIDANCE_RECEIVED` + `AGENT_MESSAGE` 중 "repaired" 패턴)

### 5-F. ContextInjectionInput

**Jordan:**

실행 중 추가 컨텍스트를 주입하는 입력창 — SessionView 하단 고정 (채팅 입력창 형태)

```
┌─────────────────────────────────────────────────────────────┐
│ 실험 중 추가 정보 입력...        Phase: [현재 ▼]  [전송]  │
└─────────────────────────────────────────────────────────────┘
```

**새 API 엔드포인트 필요:**
```
POST /api/v1/runs/{run_id}/inject
  { context: "사용자 입력 텍스트", phase: 3 }
```

**Alex:** 백엔드에 `ContextInjectionQueue.push()` 를 호출하는 엔드포인트가 필요합니다. 아직 라우트가 없으니 `routes/interaction.py`에 추가해야 합니다.

**Sam:** Phase 선택을 `-1`(모든 Phase)로 하면 현재 Phase 이후 어디서든 소비됩니다. 프론트엔드에서 "즉시 / 다음 Phase / 모든 Phase" 옵션을 드롭다운으로 주면 좋겠습니다.

---

## 6. 모던 프론트엔드 트렌드 적용

### 6-A. React 19 패턴 활용

**Jordan:**

**`use()` 훅으로 비동기 데이터 처리:**
```tsx
// 변경 전 — useEffect + useState
const [status, setStatus] = useState(null);
useEffect(() => {
  fetch(`/api/v1/research/${runId}/status`).then(r => r.json()).then(setStatus);
}, [runId]);

// 변경 후 — React 19 use()
const statusPromise = fetchRunStatus(runId);
const status = use(statusPromise);  // Suspense와 함께 사용
```

**`useTransition`으로 Guidance 제출 시 UI 버벅임 방지:**
```tsx
const [isPending, startTransition] = useTransition();
const handleGuidanceSubmit = (payload) => {
  startTransition(async () => {
    await postGuidance(runId, payload);
    setGuidancePayload(null);
  });
};
```

**`useOptimistic`으로 ProposalSheet "실행하기" 즉각 피드백:**
```tsx
const [optimisticState, addOptimistic] = useOptimistic(proposals, (state, accepted) =>
  state.map(p => p.id === accepted.id ? { ...p, status: 'queued' } : p)
);
```

### 6-B. Tailwind v4 활용

**Jordan:**

Tailwind v4는 `@layer`, CSS 변수 기반으로 완전히 전환됐습니다. 현재 코드베이스가 v4를 쓰고 있으니 활용할 수 있습니다.

```css
/* globals.css — 새 색상 토큰 추가 */
@theme {
  --color-terminal-bg: oklch(12% 0 0);
  --color-terminal-text: oklch(75% 0.15 145);   /* green-400 */
  --color-phase-active: oklch(65% 0.18 260);     /* blue */
  --color-phase-done: oklch(70% 0.14 145);       /* green */
  --color-guidance-accent: oklch(75% 0.18 55);   /* amber */
  --color-failure: oklch(62% 0.22 25);           /* red */
}
```

`color-mix()` 활용으로 투명도 처리:
```tsx
// className="bg-[color-mix(in_oklch,var(--color-phase-active)_15%,transparent)]"
// → 활성 Phase 배경의 얕은 틴트
```

### 6-C. Framer Motion v12 애니메이션

**Jordan:**

PreflightFlow 카드 전환:
```tsx
<AnimatePresence mode="wait">
  <motion.div
    key={currentQuestion.key}
    initial={{ opacity: 0, x: 40 }}
    animate={{ opacity: 1, x: 0 }}
    exit={{ opacity: 0, x: -40 }}
    transition={{ type: "spring", stiffness: 380, damping: 30 }}
  >
    <PreflightCard question={currentQuestion} />
  </motion.div>
</AnimatePresence>
```

Phase stepper 진행 표시:
```tsx
// layout 애니메이션으로 phase 전환 시 stepper 자연스럽게 이동
<motion.div layoutId="phase-indicator" className="..." />
```

TerminalPane 새 줄 추가:
```tsx
// 가상 스크롤 사용하므로 Framer Motion 대신 CSS transition만 사용
// (가상화된 DOM에 motion 컴포넌트 적용 시 성능 문제)
```

### 6-D. AI-Native UX 패턴 (2025 트렌드)

**Jordan:**

장기 실행 AI 작업의 UX에서 현재 가장 많이 채택되는 패턴들입니다.

**① Ambient Progress** — 전체 화면을 점령하지 않고 상태를 상시 표시
- RunStatusRibbon이 이 역할
- 사용자가 다른 탭을 봤다 돌아와도 즉시 파악 가능

**② Progressive Disclosure** — 필요할 때만 상세 정보 표시
- Phase stepper: 완료된 Phase는 접혀 있고, 현재 Phase만 펼쳐짐
- TokenBudgetBar: 80% 미만이면 작게, 이상이면 강조

**③ Skimmable Event Log** — 긴 로그를 빠르게 훑기 가능하게
- 이벤트 타입별 색상 코딩 (이미 있음) + Phase 구분선 추가
- `exec_stdout` 줄은 기본적으로 접혀 있고 클릭하면 TerminalPane으로 이동

**④ Non-Blocking Gates** — 사용자가 답변하지 않아도 파이프라인이 멈추지 않는 경우
- PreflightClarifier: 타임아웃 → 기본값 → 계속 → UI에서는 "기본값으로 진행됨" 카드로 표시
- GuidanceGate: 모달이 아닌 Drawer → 로그를 보면서 판단 후 응답 가능

**Sam:** Non-blocking gates 개념이 정확히 저희가 하이브리드 설계에서 원한 거예요. 사용자가 자리를 비워도 파이프라인이 멈추지 않고, 돌아왔을 때 맥락을 파악하고 개입할 수 있는.

---

## 7. API 레이어 변경사항

### 7-A. `types.ts` 확장

```typescript
// 기존 EventType에 추가
export type EventType =
  | 'SYSTEM_START' | 'SYSTEM_END'
  | 'PHASE_START' | 'PHASE_COMPLETE'
  | 'AGENT_MESSAGE' | 'AGENT_THINKING'
  | 'TOOL_CALL' | 'TOOL_RESULT'
  | 'FILE_CREATED' | 'CODE_BLOCK'
  | 'EXPERIMENT_START' | 'EXPERIMENT_RESULT'
  | 'PLAN_AWAITING_APPROVAL'
  | 'USER_GUIDANCE_NEEDED' | 'USER_GUIDANCE_RECEIVED'
  | 'SECTION_DRAFT_DONE'
  // ── Sprint 1-4 신규 ────────────────────────
  | 'PREFLIGHT_QUESTION' | 'PREFLIGHT_ANSWERED'
  | 'exec_stdout'
  | 'token_budget_warning' | 'token_budget_snapshot'
  | 'failure_escalation'
  | 'extension_proposals';

// 신규 Payload 타입
export interface PreflightPayload {
  run_id: string;
  question_key: string;
  question: string;
  default: string;
  timeout_secs: number;
  options: string[];
}

export interface TokenBudgetPayload {
  used: number;
  budget: number;
  ratio: number;
  label?: string;
}

export interface FailureEscalationPayload {
  pattern_summary: string;
  kind: string;  // "oom" | "timeout" | "runtime" | ...
}

export interface ExtensionProposalPayload {
  proposals: string[];
  exec_success: boolean;
  metrics: Record<string, unknown>;
}
```

### 7-B. `api.ts` 신규 함수

```typescript
// 컨텍스트 주입
export async function injectContext(
  runId: string,
  context: string,
  phase: number = -1
): Promise<void> {
  await fetch(`/api/v1/runs/${runId}/inject`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ context, phase }),
  });
}

// ExtensionProposer — 제안으로 새 run 시작 (임시: /research 재사용)
export async function acceptExtensionProposal(
  runId: string,
  proposal: string,
  originalTopic: string
): Promise<{ run_id: string }> {
  const res = await fetch('/api/v1/research', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      topic: originalTopic,
      goal: `[Extension] ${proposal}`,
    }),
  });
  return res.json();
}
```

### 7-C. 백엔드 신규 라우트 (`routes/interaction.py` 추가)

**Alex:** 다음 두 엔드포인트를 `routes/interaction.py`에 추가해야 합니다.

```python
@router.post("/runs/{run_id}/inject")
async def inject_context(run_id: str, body: InjectRequest):
    """ContextInjectionQueue에 사용자 컨텍스트를 추가한다."""
    # InjectRequest: { context: str, phase: int = -1 }
    queue = get_injection_queue(run_id)  # 실행 중인 queue 참조
    if queue is None:
        raise HTTPException(404, "Run not active")
    queue.push(body.context, phase=body.phase, source="user_ui")
    return {"status": "queued"}
```

**Sam:** `get_injection_queue(run_id)`가 실행 중인 `ContextInjectionQueue` 인스턴스를 가져와야 하므로, `PipelineOrchestrator`에 `_injection_queues: dict[str, ContextInjectionQueue]` 레지스트리를 추가해야 합니다. `_execute()` 시작 시 등록, 완료 시 제거합니다.

---

## 8. 구현 Sprint 계획

### FE Sprint 1: 타입 + 기존 컴포넌트 수정 (백엔드 Sprint 1-4와 병렬)

**목표**: 새 이벤트가 와도 오류 없이 렌더링

| 작업 | 파일 | 내용 |
|------|------|------|
| EventType 확장 | `lib/types.ts` | 6개 신규 이벤트 타입 + Payload 타입 추가 |
| 레지스트리 패턴 전환 | `components/LogEvents.tsx` | switch → `EVENT_RENDERERS` 맵 |
| Phase stepper 수정 | `components/LogView.tsx` | content 파싱 → metadata.phase 읽기 |
| `exec_stdout` 기본 렌더러 | `components/LogEvents.tsx` | `StdoutLine` 컴포넌트 (임시 — 텍스트만) |
| 새 이벤트 기본 처리 | `components/LogEvents.tsx` | 나머지 4개 이벤트 `DefaultEvent`로 표시 |
| guidance 라우팅 | `pages/SessionView.tsx` | preflight_ 분기 추가 |

### FE Sprint 2: 신규 컴포넌트 구현

**목표**: 5개 신규 컴포넌트 + API 함수 완성

| 작업 | 파일 | 내용 |
|------|------|------|
| PreflightFlow | `components/PreflightFlow.tsx` | 카드 + 카운트다운 + 응답 |
| TerminalPane | `components/TerminalPane.tsx` | 가상 스크롤 + 자동 스크롤 잠금 |
| TokenBudgetBar | `components/TokenBudgetBar.tsx` | Progress + 경고 |
| ProposalSheet | `components/ProposalSheet.tsx` | Bottom drawer + 카드 목록 |
| RunStatusRibbon | `components/RunStatusRibbon.tsx` | 상단 리본 + 경과 시간 |
| ContextInjectionInput | `components/ContextInjectionInput.tsx` | 하단 입력창 |
| GuidanceDrawer 전환 | `components/GuidancePanel.tsx` | 배너 → Drawer |
| `injectContext()` | `lib/api.ts` | POST /runs/{id}/inject |
| `acceptExtensionProposal()` | `lib/api.ts` | POST /research (임시) |

### FE Sprint 3: SessionView 통합 + 폴리싱

**목표**: SessionView에 모든 신규 컴포넌트 통합 + 반응형 + 접근성

| 작업 | 파일 | 내용 |
|------|------|------|
| SessionView 리팩토링 | `pages/SessionView.tsx` | 신규 컴포넌트 배치, 상태 관리 정리 |
| Framer Motion 애니메이션 | 전체 | PreflightFlow 전환, Phase stepper |
| 반응형 대응 | 전체 | Drawer가 모바일에서는 Bottom sheet로 전환 |
| 다크모드 색상 | `globals.css` | 신규 CSS 토큰 추가 |
| 접근성 | 전체 | Radix ARIA 속성, 키보드 네비게이션 확인 |
| 재진입 복원 | `pages/SessionView.tsx` | 마운트 시 guidance_status / approval_status 조회 |

---

## 9. 컴포넌트 파일 구조 (최종)

```
client/src/
├── components/
│   ├── ApprovalDialog.tsx           # 기존 — 레이아웃 개선
│   ├── GuidancePanel.tsx            # Drawer 전환
│   ├── LogView.tsx                  # Phase stepper 개선
│   ├── LogEvents.tsx                # 레지스트리 패턴 전환
│   ├── DetailPanel.tsx              # 기존 유지
│   ├── AgentConversation.tsx        # 기존 유지
│   ├── Sidebar.tsx                  # 기존 유지
│   │
│   ├── PreflightFlow.tsx            # ★ 신규
│   ├── TerminalPane.tsx             # ★ 신규
│   ├── TokenBudgetBar.tsx           # ★ 신규
│   ├── ProposalSheet.tsx            # ★ 신규
│   ├── RunStatusRibbon.tsx          # ★ 신규
│   ├── ContextInjectionInput.tsx    # ★ 신규
│   └── ui/                          # Radix 기반 — 기존 유지
│
├── hooks/
│   ├── useCountdown.ts              # ★ 신규 — PreflightFlow 타임다운
│   ├── useElapsedTime.ts            # ★ 신규 — RunStatusRibbon
│   ├── useVirtualScroll.ts          # ★ 신규 — TerminalPane
│   ├── useAutoScroll.ts             # ★ 신규 — TerminalPane 자동 스크롤
│   ├── useComposition.ts            # 기존
│   ├── useMobile.tsx                # 기존
│   └── usePersistFn.ts              # 기존
│
└── lib/
    ├── api.ts                       # injectContext, acceptExtensionProposal 추가
    ├── types.ts                     # 6개 신규 이벤트 타입 추가
    ├── constants.ts                 # 신규 이벤트 색상/레이블 추가
    └── ...
```

---

## 10. 전문가 최종 검토

**Alex (백엔드):**

프론트엔드 쪽에서 가장 중요한 것 두 가지입니다.

첫째, `PREFLIGHT_QUESTION` 타임아웃 처리. 백엔드는 60초 후 기본값으로 진행하지만 `PREFLIGHT_ANSWERED` 이벤트를 보내 사용자에게 알립니다. 프론트엔드는 이 이벤트로 카드를 "기본값 사용됨" 상태로 전환하면 됩니다. 타임아웃을 프론트엔드가 독립적으로 계산하면 네트워크 지연으로 인해 서버 실제 타임아웃과 달라질 수 있으니, UI 카운트다운은 시각적 힌트 용도로만 사용하고 실제 상태는 항상 이벤트 기준으로 처리하세요.

둘째, `ContextInjectionQueue` — 현재 `_execute()` 내부 로컬 변수라 API에서 접근이 안 됩니다. `PipelineOrchestrator._active_queues: dict[str, ContextInjectionQueue]` 딕셔너리를 추가하고 `_execute()` 시작 시 등록 / 완료 시 제거해야 합니다. 이게 없으면 inject API를 만들어도 연결할 대상이 없습니다.

**Sam (에이전트 워크플로우):**

장기 실행 관점에서 추가하면, `RunStatusRibbon`에 **세션 재진입** 시나리오도 처리해야 합니다. 사용자가 브라우저를 닫았다가 다시 열었을 때:

1. `GET /api/v1/research/{run_id}/status` → 현재 상태 확인
2. `GET /api/v1/research/{run_id}/stream` → SSE 재연결 (이미 지원)
3. `GET /api/v1/runs/{run_id}/guidance_status` → 대기 중 gate 확인 후 Drawer 자동 열기
4. `GET /api/v1/runs/{run_id}/approval_status` → 대기 중 approval 확인 후 Dialog 자동 열기

이 네 가지를 `SessionView` 마운트 시 순서대로 처리하면 완벽한 재진입이 됩니다.

**Jordan (프론트엔드):**

종합하면 이번 작업의 핵심은 두 가지입니다.

**① 파이프라인이 무엇을 하고 있는지 사용자가 항상 알 수 있어야 합니다.** 실험이 몇 시간씩 돌아가는 동안 "어디쯤 왔는지", "내 개입이 필요한지", "문제가 생겼는지"를 화면에 항상 표시하는 것이 목표입니다. `RunStatusRibbon` + `TokenBudgetBar` + `FailureEscalationAlert`가 이 역할을 합니다.

**② 사용자 개입이 필요할 때 최대한 맥락을 잃지 않게 해야 합니다.** 전체 화면 모달로 막는 대신, 옆에서 Drawer가 열리면서 로그를 보면서 판단할 수 있어야 합니다. `GuidanceDrawer` 전환과 `PreflightFlow`의 반투명 오버레이가 이 역할을 합니다.

기술적으로 React 19와 Tailwind v4를 이미 쓰고 있으니 추가 의존성은 `@tanstack/react-virtual` 하나뿐입니다. 기존 코드베이스 구조가 잘 잡혀 있어서 Sprint 1-3를 각각 1주씩 잡으면 충분합니다.

---

## 부록: 검증 체크리스트

### FE Sprint 1 완료 기준
- [ ] `exec_stdout` 이벤트가 로그에 `StdoutLine`으로 렌더링됨
- [ ] `PREFLIGHT_QUESTION` 이벤트가 `PreflightCard`로 분기됨 (UI 완성 전 placeholder 허용)
- [ ] `failure_escalation` 이벤트가 에러 없이 `DefaultEvent`로 렌더링됨
- [ ] Phase stepper가 content 파싱 없이 `metadata.phase`로 동작함

### FE Sprint 2 완료 기준
- [ ] PreflightFlow: 카운트다운 + 응답 제출 + 기본값 표시
- [ ] TerminalPane: 1000줄 이상 stdout에서 스크롤 jank 없음 (60fps)
- [ ] TokenBudgetBar: ratio에 따른 색상 변화 + 툴팁
- [ ] ProposalSheet: proposals 수신 → sheet 표시 → 제안 클릭 → 새 run 생성

### FE Sprint 3 완료 기준
- [ ] 브라우저 탭 닫고 재진입 시 Drawer 자동 복원
- [ ] 모바일 (375px)에서 Drawer가 Bottom sheet로 전환
- [ ] 키보드만으로 Preflight 응답 + Guidance 제출 가능
- [ ] Lighthouse 접근성 점수 90 이상
