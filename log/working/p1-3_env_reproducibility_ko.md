# P1-3 — 실행 환경 선택 + 재현성 기록 (구현) / P2-5 — 보류 결정

작성일: 2026-07-27 · 대상: `crewai_prototype/`, `research_system_ui/`

## 쉬운 설명 (한눈에)
비유하면 이 시스템은 "AI 연구원에게 주제만 주면 알아서 실험하고 논문까지 쓰는지"를 보는 실험대다.

- **왜 P2-5를 뺐나:** P2-5는 "AI가 만든 계획 A/B 중 사람이 고르기"였다. 그런데 이 시스템의 핵심은
  *AI가 스스로 얼마나 잘하나*를 보는 것. 중간에 사람이 계획을 고르면 "AI 실력"이 아니라
  "사람의 선택"을 측정하게 되고, 매번 결과가 달라져 **공정 비교·재현이 깨진다**. 그래서 보류했다.
  (사람 개입은 맨 처음 실험 설정 단계에만 두는 게 이 시스템의 설계 취지다.)
- **P1-3에서 한 일 = "실험 주방 기록기":** 실험을 어떤 파이썬 환경(=주방)에서, 어떤 라이브러리
  버전(=재료)으로 돌렸는지를 **자동으로 결과에 적어둔다**(torch 2.5.0, crewai 1.14.3 … + 지문 코드).
  그래야 나중에 "똑같이 다시 해봐" 할 때 재현이 된다. 재현성은 이 연구의 1순위 지표라 꼭 필요했다.
- **실험 시작 화면에 "실행 환경 고르기"를 추가:** 시스템이 컴퓨터에 설치된 파이썬 환경들 중
  **실험에 필요한 재료가 다 갖춰진 것만 골라 보여주고**(추천 자동 선택), 사용자는 목록에서 고르거나
  경로를 직접 넣을 수 있다. 고른 환경 정보는 위의 "주방 기록"에 그대로 남는다.

즉, **P2-5는 연구 공정성 때문에 접고, P1-3은 "실험 환경 선택 + 자동 재현성 기록"으로 완성**했다.

## 배경 — "연구 파이프라인 본질"에 맞춘 재검토
UI 개선 캠페인의 P1-3(환경 선택)·P2-5(대안 계획 택1)는 상용 AI agent UI 조사에서 파생돼
*제품 사용성* 렌즈가 강했다. 구현 전, MARS가 **통제된 벤치마크**(독립변수=프레임워크,
HITL=측정 대상, 재현성=1차 지표)라는 본질에 부합하는지 재검토했다.

시스템 본래 목적(사용자 확인, 2026-07-27): **처음 실험 configure만 문답으로 확정하고 나면,
이후엔 AI가 최적 실험을 알아서 완주·문서화하는지를 측정**하는 것. 즉 인간 개입은 *앞단 설정*에
집중하고, 파이프라인 중간에 인간이 조종하는 요소는 벤치마크를 오염시킨다.

### P2-5 (대안 계획 인간 택1) — **보류**
- 인간이 계획을 택1하면 run 결과가 프레임워크의 자율 플래닝이 아니라 인간 선택에 좌우 → **독립변수 오염**.
- 인간 선택은 비결정적 → **재현성(1차 지표) 저하**.
- 프레임워크별 대안 생성 능력 차이로 인간 개입력이 불균등 → **공정성(동일 개입 프로토콜) 위반**.
- 현재 planner는 단일안 → 복수안 생성은 3 프레임워크 동일 적용해야 하는 **프로토콜 변경**(스코프 큼).
- (원한다면 통제·로깅되는 HITL ablation 조건으로 재정의 시 RQ4에 기여 가능 — 실험설계 결정 사안.)

### P1-3 (환경 선택) — **재현성 기록형으로 구현**
- 벤치마크에서 실행 환경 = 재현성 변수 → "설정 시점 1회 선택 + result 메타데이터 기록 + run 표시"로 재프레임.
- 시스템 목적("config는 시작에만")과 부합. 프레임워크별 env(crewai/autogen/langgraph)와 연결 가능.

## 구현 내용

### 공유 모듈 `core/env_detect.py` (신규)
능력 기반(필수 패키지 import 가능 여부) 환경 탐지 + 재현성 메타데이터. **stdlib-only, import 부작용 없음**
(깨진/base 환경에서도 안전). 헤드리스 러너·웹 API·phase3가 공유한다.
- `REQUIRED_MODULES = [crewai, torch, sklearn, pandas, dotenv]`
- `list_suitable_environments()` — 현재 인터프리터 + conda 후보들을 검사해 **적합한** 것만 반환(추천=현재 우선).
  - **find_spec 선검사(설치 여부, 빠름) → 통과분만 실제 `__import__`(깨진 DLL 탐지)** 2단계.
  - 시스템에 conda env가 많을 수 있어(실측 25개) **ThreadPoolExecutor 병렬 프로빙**.
- `describe_python(py)` — python/env 이름·버전·platform·패키지 버전(torch/crewai/numpy 등) 수집(서브프로세스, importlib.metadata).
- `requirements_hash(packages)` — 패키지 버전 집합의 sha256 앞 12자(재현성 지문).
- `harness/anchor_run.py`는 이 모듈의 프리미티브를 재사용(중복 제거).

### 백엔드 재현성 기록 (P1-3a)
- `runtime/models.py`: 이벤트 타입 `EXECUTION_ENVIRONMENT` 추가.
- `core/handoff_models.py`: `ExecutorResult.environment: dict` 필드 추가.
- `phases/phase3_execution.py`: Phase 3 시작 시 `describe_python(선택 python)`으로 재현성 블록 구성
  (env/버전/패키지/device/seed/entry_command/requirements_hash) → **`EXECUTION_ENVIRONMENT` 이벤트 emit**
  + 성공 시 `ExecutorResult.environment`에 첨부. 실험 명령 구성은 `_experiment_cmd()`로 공용화(entry_command 일치).
- `orchestration/pipeline_orchestrator.py`: `result_summary["environment"]`에 적재(완료 run의 GET /result·표시용).

### 백엔드 환경 선택 (P1-3b/d)
- `api/routes/environments.py` (신규): `GET /api/v1/environments` — 적합 env 목록(10분 캐시, `?refresh=true`).
- `api/app.py`: 라우터 등록. `api/schemas.py`: `ResearchRequest.environment`(선택 python 경로) 추가.
- 선택 env 흐름: ResearchRequest.environment → `session.metadata` → orchestrator가 Phase 3에 `python_exe` 전달
  → `_run_script`가 그 python으로 실험 subprocess 실행(파일 없으면 현재 인터프리터로 폴백 + 경고).

### 프론트엔드 (P1-3c)
- `lib/api.ts`: `getEnvironments()`, `EnvironmentOption`/`EnvironmentListResponse`, `StartResearchRequest.environment`.
- `lib/types.ts`: `EventType`에 `EXECUTION_ENVIRONMENT`, `ReproEnvironmentPayload`.
- `pages/Dashboard.tsx`: 고급 설정에 **실행 환경(재현성) 선택** — VS Code 인터프리터 피커식 라디오 목록
  (추천 pre-select·현재 배지·Python 버전·경로) + **"직접 경로 입력" 폴백** + **적합 0개 Empty State**.
  다이얼로그 오픈 시 조회, 대시보드 진입 시 캐시 프리워밍.
- `components/LogEvents.tsx` + `lib/constants.ts`: `EXECUTION_ENVIRONMENT` 전용 카드(env/Python/device/지문/핵심 패키지 칩/entry_command) + 아이콘/라벨.

## 검증
- 프론트: `npm run check`(tsc --noEmit) **exit=0, 0 errors**.
- 백엔드: 변경 9파일 AST OK + orchestrator/phase3/route import 체인 OK.
- env 목록 API 실호출: 후보 25개 → **MARS 1개만 적합(추천·현재), 32.7s(첫 호출, 이후 10분 캐시)**.
- 라이브 파이프라인 run으로 `EXECUTION_ENVIRONMENT` 이벤트 실제 발생 확인(진행 로그 참조).

## 관련 문서
- `docs/ui_option_selection_campaign_ko.md` — 상위 UI 캠페인(P0/P1/P2 전체).
- `docs/env_and_provider_changes_ko.md` — 실행 환경 가드/네이티브 프로바이더.
