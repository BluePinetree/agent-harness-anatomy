# Research System V4 — 설계 회의 결론

> 참여: 30년 경력 시스템 엔지니어 (Engineer) + 30년 경력 알고리즘 전문가 (Algorithm)  
> 주제: 서킷 브레이커 없는 신뢰성 높은 연구 자동화 파이프라인 재설계  
> 날짜: 2026-05-19

---

## 현재 시스템의 근본 문제

```
문제: 서킷 브레이커 (_MAX_CONSECUTIVE_REPAIR_FAILURES = 3)
증상: 코드 수정 3회 실패 → 즉시 중단 → "Code could not be fixed" 메시지
원인: 실패가 고착화될 거라는 가정으로 무한 루프를 방지하려 했으나,
      실제로는 사용자 개입 없이 조용히 포기하는 시스템이 됨
```

---

## 핵심 설계 결론 (Engineer + Algorithm 합의)

### 1. 서킷 브레이커 → 에스컬레이션 루프로 교체

**기존:**
```python
if repair_count >= _MAX_CONSECUTIVE_REPAIR_FAILURES:
    emit("Circuit breaker: aborting")
    break  # 포기
```

**신규:**
```python
while True:  # 절대 포기 안 함
    attempt_count += 1
    repair_result = coder_agent.repair(file, error, hint)
    if check_passes(file):
        break
    if attempt_count >= MAX_AUTO_ATTEMPTS:
        # 에스컬레이션: 사용자에게 묻는다
        hint = await ask_user_for_guidance(file, error, attempts_log)
        attempt_count = 0  # 사용자 힌트 받았으면 카운터 리셋
```

### 2. 사용자 승인 게이트 (Phase 1 → Phase 2 전환 시)

```python
# pipeline_orchestrator.py
gate = ApprovalGate(plan_payload=plan_bundle)
approval_registry.register(run_id, gate)
emit_event(PLAN_AWAITING_APPROVAL, payload=plan_bundle)

gate.event.wait(timeout=3600)  # pipeline thread blocks here

if gate.feedback:
    # 사용자가 수정 요청 → planner/designer 재실행
    return await self._replan(gate.feedback)
# 승인됨 → 코딩 진행
```

**API 엔드포인트:**
- `POST /runs/{id}/approve` → `{"action": "approve"}`
- `POST /runs/{id}/approve` → `{"action": "reject", "feedback": "..."}`
- `POST /runs/{id}/approve` → `{"action": "modify", "feedback": "..."}`

### 3. 단계별 코딩 (Stage 1 → 2 → 3)

Designer가 `stage_assignments: dict[str, int]`를 출력함:
- Stage 1: config, utils, constants (의존성 없는 파일)
- Stage 2: data loader, model, trainer (Stage 1 의존)
- Stage 3: experiment_impl.py (모든 것의 진입점)

각 Stage 후:
1. `python -c "import ast; ast.parse(open(f).read())"` — 구문 검사
2. `python -c "import importlib.util; ..."` — 임포트 검사
3. 실패 시 수리 루프 → 한도 초과 시 사용자 에스컬레이션

### 4. 섹션별 논문 작성

**작성 순서 (실험→요약 방향):**
```
Experiments → Introduction → Related Works → Proposed Method → Conclusion → References → Abstract
```
*(Experiments를 먼저 써야 나머지 섹션이 실제 수치를 참조할 수 있음)*

**자체 검증 기준 (SelfVerifier):**
| 항목 | 가중치 | 기준 |
|------|--------|------|
| 실험 결과 참조 | 35% | 실제 metric 수치가 언급되었는가 |
| 단어 수 | 30% | 섹션별 최소 단어 수 충족 |
| 인용 형식 | 20% | [1] 스타일 참조 유효 |
| 마크다운 문법 | 15% | 코드블록, 표 등 올바른 형식 |

품질 점수 < 0.7 → 최대 3회 재작성 → 여전히 미달 시 `NEEDS_REVIEW` 플래그 달고 진행

---

## 신규 시스템 아키텍처 요약

```
Phase 0: Workspace Setup
  └─ 사용자 지정 경로 또는 outputs/{timestamp}_{slug}/

Phase 1: Planning & Design  [사용자 승인 게이트]
  ├─ PlannerAgent  →  PlannerResult (JSON)
  ├─ DesignerAgent  →  DesignerResult (파일트리 + AST + stage 할당)
  └─ ApprovalGate  →  사용자가 APPROVE/REJECT/MODIFY 응답할 때까지 대기

Phase 2: Staged Coding  [에스컬레이션 루프, 절대 포기 없음]
  ├─ Stage 1: config/utils 작성 + 구문/임포트 검사
  ├─ Stage 2: data/model/trainer 작성 + 구문/임포트 검사
  └─ Stage 3: experiment_impl 작성 + smoke test

Phase 3: Experiment Execution
  ├─ entry_point 스크립트 실행
  ├─ result.json 캡처
  └─ 실패 시 → 에스컬레이션 루프 (서킷 브레이커 없음)

Phase 4: Section-by-Section Writing
  ├─ Experiments → Introduction → Related Works → ...
  ├─ 섹션별 자체 검증 (SelfVerifier)
  └─ 최종 통합 검증 + 약한 섹션 재작성
```

---

## 제거되는 것

| 기존 구성요소 | 이유 |
|--------------|------|
| `_MAX_CONSECUTIVE_REPAIR_FAILURES = 3` | 에스컬레이션으로 대체 |
| `research_coordinator_v3.py` Phase 2 circuit breaker | 전면 재설계 |
| 단일 FileCoder Crew (파일 전체 한번에) | Stage별 Crew로 분리 |
| Writer 단일 태스크 | 섹션별 태스크 7개로 분리 |

---

## 구현 우선순위

1. `platform/constants.py` — 모든 설정값 중앙화
2. `orchestration/approval_registry.py` — 승인 게이트 메커니즘
3. `phases/phase2_coding.py` — 에스컬레이션 루프 (핵심)
4. `orchestration/pipeline_orchestrator.py` — 전체 파이프라인 조율
5. `phases/phase1_planning.py` + `phase4_writing.py`
6. API 엔드포인트 수정 (`/approve`, `/guidance`)

---

## 관련 문서

- [ARCHITECTURE.md](ARCHITECTURE.md) — 모듈 설계, 기술 스택
- [API_SPEC.md](API_SPEC.md) — REST/SSE 명세
- [MODULE_SPEC.md](MODULE_SPEC.md) — 클래스/함수 시그니처
- [PIPELINE_SPEC.md](PIPELINE_SPEC.md) — Phase별 상태 머신
- [ERROR_RECOVERY_SPEC.md](ERROR_RECOVERY_SPEC.md) — 에러 분류 및 복구
- [AGENT_SPEC.md](AGENT_SPEC.md) — 에이전트 프롬프트/스키마
- [CONSTANTS.md](CONSTANTS.md) — 모든 설정값
