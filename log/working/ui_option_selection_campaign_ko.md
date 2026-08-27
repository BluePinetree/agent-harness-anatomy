# UI 개선 캠페인 — "선택지 → 선택 → 진행" 패턴 도입

작성일: 2026-07-27 · 대상: `research_system_ui/` (Vite + React19 + TS + Tailwind + Radix)
근거: `docs/s2_expert_panel_review_ko.md`(UI 검토) + 상용 AI agent UI 자료조사(2 전문가)

## 회의 종합 (설계 원칙)
상용 AI agent(Claude Code plan mode·Cursor·Copilot Workspace·Devin·Manus·v0 등)와 개발도구(VS Code 인터프리터 선택·모델 드롭다운·W&B/AutoTrain)를 조사한 결과, 모든 선택 지점의 공통 처방:
1. **추천안을 pre-highlight한 옵션 카드/칩(택1)** — 라디오 ≤6개, "(추천)" 라벨
2. **"직접 입력" 폴백** 항상 제공
3. **비가역·고비용 액션엔 2차 확인**(HITL "무엇이 잘못될 수 있나")
4. 현재 상태 상시 표시, 자유입력 최소화(재현성·표기일관성), WCAG(role=radio·키보드)

핵심 발견(검토): 백엔드는 이미 상호작용 이벤트에 `options` 필드를 실어 보내나 **프론트가 렌더링 0건**이었고, UI는 대부분 자유입력이었다.

## 구현 완료 (tsc `npm run check` exit=0, 0 errors)

### P0-1 — Preflight 선택지형 ✅
- 백엔드 `crewai_prototype/orchestration/preflight_clarifier.py`: `PreflightQuestion.choices` 필드 추가 + 기본 4문항에 후보 답변 부여(첫 항목=권장) + `PREFLIGHT_QUESTION` 이벤트 metadata에 `choices` 포함.
- 프론트 `lib/types.ts`(PreflightPayload.choices), `pages/SessionView.tsx`(choices 파싱), `components/PreflightFlow.tsx`: 자유입력 textarea → **선택지 라디오 칩(추천 pre-select) + "직접 입력…" 폴백**. "기본값"→"추천" 라벨. 60초 타임아웃 유지.

### P0-2 — 초기 config 선택형 ✅ (`pages/Dashboard.tsx`)
- **선호 프레임워크**: 자유텍스트("PyTorch, scikit-learn") → **ToggleGroup 다중선택**(PyTorch/scikit-learn/XGBoost/LightGBM/timm/TensorFlow). 제출 로직(comma 문자열)은 유지 → 표기 오타 방지.
- **연구 분야**: 자유입력 → **프리셋 칩(컴퓨터 비전/정형 데이터/시계열/NLP) + 직접 입력 폴백**.

### P1-4 — 수리 가이던스 개선 ✅ (`components/GuidanceDrawer.tsx`)
- `payload.options` 중 액션 동사를 제외한 "의미 있는 수리 제안"을 **선택 칩(클릭 시 힌트로 채움)**으로 렌더(백엔드가 fix 후보를 보낼 때 자동 노출).
- **건너뛰기 경고** 상시 표시: "건너뛰면 의존 파일도 실패할 수 있습니다"(HITL).

## P1-3 — 실행환경(venv) 선택 UI ✅ **구현됨 (재현성 기록형, 2026-07-27)**
연구 파이프라인 본질(재현성=1차 지표) 재검토 후, "편의용 per-run 토글"이 아니라
**"설정 시점 1회 선택 + result 재현성 메타데이터 기록 + run 표시"**로 재프레임해 구현.
- 신규 `core/env_detect.py`(능력 기반 탐지, find_spec 선검사+병렬), `GET /api/v1/environments`,
  `ResearchRequest.environment`, phase3 `EXECUTION_ENVIRONMENT` 이벤트 + `ExecutorResult.environment`,
  Dashboard 인터프리터 피커(추천 pre-select+직접입력 폴백+Empty State), 전용 로그 카드.
- 상세·검증: `docs/p1-3_env_reproducibility_ko.md`.

## P2-5 — 대안 계획 택1 ⛔ **보류 (벤치마크 타당성, 2026-07-27)**
- 설계(Copilot Workspace/Antigravity): 대안 계획 A/B 카드 택1 + 단계별 인라인 코멘트.
- **보류 이유(제품 렌즈 ≠ 연구 렌즈):** 인간 계획 택1은 독립변수(프레임워크 자율성)를 오염,
  재현성(1차 지표) 저하, 프레임워크별 개입력 불균등(공정성 위반). 시스템 목적("config는 시작에만,
  이후 자율")과도 상충. planner 복수안 생성은 3 프레임워크 동일 적용해야 하는 프로토콜 변경(스코프 큼).
- (원하면 통제·로깅되는 HITL ablation 조건으로 재정의 시 RQ4 기여 가능 — 실험설계 결정.)
- 상세: `docs/p1-3_env_reproducibility_ko.md`.

## 실무 주의(반영/권고)
- 접근성: 칩은 `role="radio"`/`aria-checked`로 구현(키보드). 파괴적 액션을 default 버튼에 두지 않음.
- 타임아웃: 자동 진행 기본값을 시각적으로 pre-highlight("N초 후 추천값으로 진행"). 60초 preflight엔 "시간 연장" 옵션 권고(WCAG).
- 다음: P1-3/P2-5 착수 시 위 설계대로 백엔드 이벤트/계획생성부터.
