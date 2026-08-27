# 자율 연구 시스템 — 프레임워크 비교 연구 노트

> **작성일**: 2026-04-14  
> **상태**: 진행 중  
> **목표**: 동일한 연구 자동화 워크플로우를 세 가지 멀티에이전트 프레임워크로 구현하고 비교 분석한다.

---

## 1. 연구 동기

멀티에이전트 오케스트레이션 프레임워크는 급속히 발전하고 있으나, 실제 연구 자동화 파이프라인에서의 성능·구조·운용성 비교 자료가 부족하다. 본 프로젝트는 **동일한 문제**에 대해 CrewAI, AutoGen, LangGraph 세 가지 프레임워크를 각각 구현함으로써 실증적 비교 기반을 마련한다.

**핵심 질문**:
- 각 프레임워크는 어떤 제어 모델을 따르는가?
- 동일 워크플로우에서 구현 복잡도와 유연성은 어떻게 다른가?
- 프로덕션 환경에 가장 적합한 프레임워크는 무엇인가?

---

## 2. 시스템 개요

### 공통 파이프라인

모든 구현체는 아래 6단계 파이프라인을 동일하게 수행한다.

```
Planner → Designer → Coder → Executor → Analyzer → Writer
```

| 단계 | 역할 |
|------|------|
| **Planner** | 연구 주제 분석, 5단계 계획 수립 |
| **Designer** | 실험 설계, 가설 및 방법론 정의 |
| **Coder** | 실험 코드 생성 |
| **Executor** | 코드 실행 (샌드박스/컨테이너) |
| **Analyzer** | 결과 분석, 수렴 여부 판단 |
| **Writer** | 최종 연구 보고서 작성 |

### 프로젝트 구조

```
research_system/
├── crewai_prototype/      # CrewAI 구현체 (프로덕션 권장)
├── autogen_prototype/     # AutoGen 구현체
├── langgraph_prototype/   # LangGraph 구현체
└── research_system_ui/    # 통합 React 대시보드
```

---

## 3. 구현체별 아키텍처

### 3-1. CrewAI — 역할 기반 순차 실행

**제어 모델**: 고정 순서의 크루(Crew) + 콜백  
**성숙도**: 가장 높음 (프로덕션 권장)

CrewAI 구현체는 두 레이어로 구성된다.

```
FastAPI / CLI (main.py)
        ↓
ResearchCrew (crew.py)
        ↓
  ┌─────────────────────────────────┐
  │  Planner → Designer → Coder    │
  │       ↓ (반복 루프, 최대 3회)   │
  │  Executor → Analyzer → Writer  │
  └─────────────────────────────────┘
```

**특징**:
- **워크스페이스 스캐폴딩**: 각 실행(run)마다 격리된 디렉터리 생성 → 실행 간 오염 방지
- **런 컨트랙트**: 출력 스키마(진입점, 결과 구조)를 명시적으로 강제
- **패치 실행**: 파일 수정 시 감사 로그 포함
- **우아한 성능 저하**: E2B 미설치 시 로컬 subprocess, MLflow 미설치 시 로컬 JSON으로 자동 전환

**도구 연동**:

| 에이전트 | 도구 |
|----------|------|
| Planner / Designer | ChromaDB (RAG) |
| Coder | ChromaDB + 파일 I/O |
| Executor | E2B 샌드박스 + MLflow |
| Analyzer | MLflow + ChromaDB |
| Writer | ChromaDB + 파일 I/O |

---

### 3-2. AutoGen — 동적 그룹 채팅

**제어 모델**: SelectorGroupChat (동적 발언자 선택)  
**성숙도**: 베타

고정 순서 없이 에이전트가 협의하며 방향을 결정한다.

```
          SelectorGroupChat
                 │
   ┌─────────────┼─────────────┐
   │             │             │
ResearchPlanner  Coder       Critic
                 │
              Executor
         (LanceDB + Logger)
```

**특징**:
- 고정 워크플로우 없음 — 에이전트가 맥락에 따라 다음 발언자 결정
- 최대 30라운드 대화, `RESEARCH_COMPLETE` 신호로 종료
- 탐색적·적응형 문제 해결에 유리

---

### 3-3. LangGraph — 명시적 상태 그래프

**제어 모델**: StateGraph (노드 + 조건부 엣지)  
**성숙도**: 베타

워크플로우가 그래프 구조로 명시되어 실행 경로를 눈으로 확인할 수 있다.

```
START
  ↓
[Planner] → [Designer] → [Coder] → [Executor]
                                       ↓ 성공?
                               Yes → [Analyzer]
                                       ↓ 목표 달성?
                               Yes → [Writer] → END
                                       ↓ No (최대 3회)
                               No  → [Coder] (디버그 루프)
                                       ↓ 실패
                               [Failure] → END
```

**핵심 혁신 — ResearchState TypedDict**:

```python
class ResearchState(TypedDict):
    research_input: ResearchInput
    session_id: str
    run_id: str
    research_plan: str
    experiment_design: str
    generated_code: str
    experiment_results: list[dict]
    best_metrics: dict
    debug_info: dict          # loop_count, max_loops
    meets_target: bool
    report: str
    phase_history: list[str]
    status: str
```

타입 안전한 상태 관리로 에이전트 간 데이터 흐름이 명확하다.

**특징**:
- 자동화된 자기 수정(self-correction): 실행 실패 시 Coder로 자동 재라우팅
- 조건부 엣지 함수(`check_execution_result`, `should_continue_or_debug`)로 분기 명시
- 미션 크리티컬 환경에 적합

---

## 4. 통합 UI

**스택**: React 19 + TypeScript + Vite + TailwindCSS + shadcn/ui

세 프레임워크 모두를 동일한 UI로 시각화한다. 백엔드 교체 시 프론트엔드 변경 불필요.

```
[Dashboard]         세션 목록 + 진행 상황
[SessionView]       3열 레이아웃 (필터 | 실시간 로그 | 상세)
[ComparisonView]    프레임워크 간 메트릭 비교 (막대/레이더 차트)
```

**디자인 테마**: Mission Control (우주 관제실 모티프)
- 배경: Deep Navy `#0A0E1A`
- 강조: Cyan `#00D4FF`
- 성공: Emerald `#10B981`
- 오류: Coral `#EF4444`

### 이벤트 타입 (12종)

| 이벤트 | UI 표현 |
|--------|---------|
| `SYSTEM_START` / `SYSTEM_END` | 상태 배너 |
| `AGENT_MESSAGE` | 컬러 말풍선 |
| `AGENT_THINKING` | 접을 수 있는 회색 블록 |
| `TOOL_CALL` / `TOOL_RESULT` | 파란/초록/빨간 카드 |
| `FILE_CREATED` | 파일 링크 |
| `CODE_BLOCK` | 신택스 하이라이트 코드 |
| `EXPERIMENT_RESULT` | 메트릭 테이블 |
| `PHASE_COMPLETE` | 구분선 + 체크마크 |

---

## 5. 공통 규약

### JSONL 로그 포맷

모든 프레임워크가 동일한 이벤트 구조를 출력한다.

```json
{
  "timestamp": "2026-04-10T12:34:56.789Z",
  "session_id": "session_abc123",
  "run_id": "run_def456",
  "event_type": "AGENT_MESSAGE",
  "agent_name": "Research Planner",
  "content": "5단계 연구 계획을 수립했습니다...",
  "metadata": {
    "phase": 1,
    "tool_name": "chromadb_search",
    "success": true
  }
}
```

### 출력 디렉터리 구조

```
outputs/{run_id}/
├── generated_code/
│   ├── experiment.py
│   └── requirements.txt
├── results/
│   ├── metrics.json
│   └── figures/
└── report.md
```

### 컨텍스트 관리 파일

| 파일 | 용도 |
|------|------|
| `context/runtime_memory.md` | 런타임 상태 기록 |
| `context/handoff_state.json` | 에이전트 간 핸드오프 데이터 |
| `context/compact_history.md` | 압축된 히스토리 |

---

## 6. 프레임워크 비교

| 항목 | CrewAI | AutoGen | LangGraph |
|------|--------|---------|-----------|
| **제어 모델** | 고정 순서 크루 | 동적 그룹 채팅 | 명시적 상태 그래프 |
| **유연성** | 낮음 (역할 고정) | 높음 (적응형) | 중간 (그래프 정의) |
| **가시성** | 중간 | 낮음 | 높음 |
| **구현 복잡도** | 높음 (monolithic) | 중간 | 낮음 (노드 단위) |
| **디버그 루프** | 수동 반복 설정 | 그룹 토론으로 해결 | 자동 조건부 재라우팅 |
| **상태 관리** | JSON 핸드오프 파일 | 인메모리 메시지 | TypedDict (타입 안전) |
| **외부 서비스** | E2B, MLflow, ChromaDB | LanceDB, OpenHands | Pinecone, Docker, W&B |
| **적합 시나리오** | 구조화된 연구 자동화 | 탐색적 문제 해결 | 미션 크리티컬 |

### 권장 사용 시나리오

- **프로덕션 배포** → CrewAI (성숙도 최고, 워크스페이스 격리)
- **유연한 실험** → AutoGen (에이전트 간 자유 토론)
- **감사 가능한 실행** → LangGraph (모든 분기 명시, 재현 가능)

---

## 7. 기술 스택

### 백엔드 (Python)

| 범주 | 기술 |
|------|------|
| 프레임워크 | CrewAI, AutoGen 0.4.x, LangGraph |
| LLM 클라이언트 | OpenAI SDK, Anthropic SDK, Google GenAI |
| RAG | ChromaDB, LanceDB, Pinecone |
| 실험 실행 | E2B 샌드박스, Docker |
| 실험 추적 | MLflow, Weights & Biases |
| API 서버 | FastAPI + uvicorn |
| 비동기 | asyncio, Celery (선택) |

### 프론트엔드 (React)

| 범주 | 기술 |
|------|------|
| 프레임워크 | React 19 + TypeScript + Vite |
| 스타일 | TailwindCSS 4 + shadcn/ui |
| 차트 | Recharts |
| 코드 하이라이트 | react-syntax-highlighter |
| 애니메이션 | Framer Motion |

### 지원 LLM

- **OpenAI**: GPT-5.2, GPT-5-mini
- **Anthropic**: Claude Sonnet 4.5, Claude Haiku 4.5
- **Google**: Gemini 2.5 Pro/Flash

---

## 8. 현재 상태

| 구성요소 | 상태 | 비고 |
|----------|------|------|
| CrewAI Prototype | 안정 | V1 레거시 코어 유지, V2 API 리팩터링 진행 중 |
| AutoGen Prototype | 베타 | 그룹 채팅 동작, 안정화 필요 |
| LangGraph Prototype | 베타 | 상태 그래프 검증 완료, 디버그 루프 확인 |
| Research UI | 안정 | 3열 레이아웃, 스트리밍, 비교 뷰 완성 |

---

## 9. 향후 과제

- [ ] AutoGen, LangGraph 구현체 안정화
- [ ] 동일 토픽으로 3개 프레임워크 동시 실행 후 정량 비교
- [ ] ComparisonView에 실행 시간, 토큰 사용량, 출력 품질 메트릭 추가
- [ ] CrewAI V2 API 리팩터링 완료 (`entrypoints/` 모듈화)
- [ ] 프레임워크별 최종 분석 보고서 작성

---

## 참고 문서

- [crewai_claude_structure_diff_ko.md](crewai_claude_structure_diff_ko.md) — CrewAI vs Claude-Code 구조 비교
- [crewai_vs_claude_structure_analysis_ko.md](crewai_vs_claude_structure_analysis_ko.md) — 상세 구조 분석
- [crewai_prototype/CLAUDE.md](crewai_prototype/CLAUDE.md) — CrewAI 구현 상세
- [autogen_prototype/CLAUDE.md](autogen_prototype/CLAUDE.md) — AutoGen 구현 상세
- [langgraph_prototype/CLAUDE.md](langgraph_prototype/CLAUDE.md) — LangGraph 구현 상세
