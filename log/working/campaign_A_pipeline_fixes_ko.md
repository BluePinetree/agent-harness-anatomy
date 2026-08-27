# Campaign A — 파이프라인/도구 수정 (S2 패널 리뷰 후속)

착수: 2026-07-24 · 근거: `docs/s2_expert_panel_review_ko.md`
목표: S2 run에서 드러난 **파이프라인/도구 레벨 결함**을 근본 수정해 이후 모든 시나리오(S1/S7…) 품질을 끌어올린다. (per-run 코드생성 결함이 아니라 repo 소스의 durable 수정)

방식: 관련 전문가(에이전트)가 각자 **서로 다른 파일 1개**를 맡아 구현·자체검증 → 오케스트레이터가 통합 검증.

## 작업 항목

| ID | 파일 | 내용 | 담당 | 상태 |
|---|---|---|---|---|
| A1 | `crew_tools/syntax_check_tool.py` | `check_dataclass_fields` cross-module 동명 dataclass 오탐 제거 (module-qualified 매핑, 동명 2개↑ 검증 제외) | Tool/정적분석 전문가 | 진행 |
| A2 | `phases/phase2_coding.py` | 수리 회귀 가드: 수리 결과가 기존 public 함수/클래스/메서드를 삭제·축소하면 반려(스텁 게이밍 차단) | 코드생성/수리 전문가 | 진행 |
| A3 | `phases/phase3_execution.py` | run_contract 검증: 실행 결과가 계획(success_criteria)·기대 규모와 불일치(예 epoch 강등, metric 부재) 시 명시 이벤트/경고. 실행규모 강등 근원 조사 포함 | 실행/오케스트레이션 전문가 | 진행 |

## 원칙
- 3개 파일이 서로 겹치지 않음 → 병렬 구현 안전.
- 각 담당은 수정 후 **AST parse + 모듈 import + 대상 결함 재현 테스트**로 자체검증.
- 기존 동작(정상 케이스) 회귀 없어야 함.

## 진행 기록 (2026-07-24 완료)

### A1 결과 — `crew_tools/syntax_check_tool.py` ✅
- 오탐 근원: (1) `@dataclass`를 **클래스 이름만으로** 수집해 동명(`MetricBundle`) 충돌 시 덮어씀, (2) valid 멤버 수집이 필드만 모으고 **메서드 미포함** → `mb.compute()`를 없는 필드로 오판.
- 수정: `_is_dataclass_def`(Call형 데코레이터 인식) + `_collect_class_members`(필드+메서드+프로퍼티+중첩클래스) 신설. 수집을 `(파일경로, 멤버집합)` 목록으로 바꿔, **정의 2개↑이면 entry의 import 힌트로 유일 특정될 때만 검증, 아니면 보수적 제외**(오탐 방지).
- 자체검증: 6케이스 통과(오탐 해소 + 진탐 3종 유지). 회귀: 상속 필드는 여전히 미포함(기존 한계, 범위 밖).

### A2 결과 — `phases/phase2_coding.py` ✅
- 추가: `_public_symbol_set`/`_noop_symbol_set`/`_is_noop_body`/`_detect_api_shrink`(순수 AST 함수) + `_apply_repair_guarded` 래퍼.
- 로직: 수리 **전** 소스 대비 수리본이 public 함수/클래스/메서드를 **삭제**하거나 **no-op 스텁화**하면 → 디스크 미기록·반려, `REPAIR_REJECTED_API_SHRINK` 이벤트, "public API 제거 금지" 규칙 붙여 1회 재생성. 재실패 시 원본 유지.
- 연결: `_repair_loop`·`_run_smoke_test`의 수리-write 2곳을 가드 경유로 교체(초기 생성/entry 생성은 비대상).
- 자체검증: (a)스텁축소 감지=반려, (b)문법만수정=통과, (c)API확장=통과. 통과.

### A3 결과 — `phases/phase3_execution.py` ✅
- **강등 근원 규명**: `_run_script`가 실행 커맨드에 **`--epochs`를 안 넘김** → scaffold `build_parser` 기본값 `--epochs=1`(builder.py:450)이 항상 사용됨. 계획이 "3 epoch"여도 CLI로 안 흘러 조용히 1로 강등. (지시대로 실행규모는 강제 변경 안 하고 **감지·표면화**만.)
- 추가: `_find_numeric`/`_has_numeric_metric`/`_planned_epochs`/`check_contract`/`_emit_contract_events`.
- 이벤트: `EXECUTION_SCALE`(항상, planned vs actual), `CONTRACT_METRICS_MISSING`(numeric metric 전무 시), `CONTRACT_VIOLATION`/`EXECUTION_SCALE_DOWNGRADE`(계획>실제 epoch). 성공/실패 판정은 미변경(리스크 회피), `metrics["contract_check"]` 요약 첨부.
- 자체검증: 5케이스 통과.

### 통합 검증 (오케스트레이터) ✅ 완료
- MARS 환경에서 3개 파일 AST parse + 통합 import + 전체 파이프라인(`entrypoints.init`) import + A1/A2/A3 신규 심볼 존재 확인.
- **E2E 회귀 검증**: S5(regression) MARS run `b9265cdc22f8` → **그린**(exec_success=true).
  - **회귀 없음**: 지표가 이전 S5 앵커(`c0dad6`)와 bit-identical (GB RMSE 0.5422/R² 0.7756, LR RMSE 0.7456/R² 0.5758). 오히려 feature_importance·success_criteria(meets_all=1.0) 등 더 풍부.
  - **A1 검증**: smoke 클린 통과(dataclass 오탐→파괴적 수리 미발생). 
  - **A2 검증**: 수리 불필요한 clean run이라 가드 정상 dormant(오작동 없음).
  - **A3 검증**: `contract_check` dict가 result metrics에 첨부(`{planned_epochs:null, actual_epochs:null, has_numeric_metric:true, violations:[]}` — tabular라 위반 없음=정상). `EXECUTION_SCALE` 이벤트 실제 발생 확인(내용: "Execution scale — planned_epochs=None, actual_epochs=None").

### 남은 소소한 항목 / 후속 결정
- **이벤트 taxonomy 등록(소)**: `EXECUTION_SCALE`/`CONTRACT_METRICS_MISSING`/`CONTRACT_VIOLATION`/`REPAIR_REJECTED_API_SHRINK`가 taxonomy 미등록이라 `normalize_event_type`이 `AGENT_MESSAGE`로 정규화함(내용은 보존, 타입별 질의는 불가). 논문 trace의 타입별 집계를 위해 `runtime/models.py`의 이벤트 상수/정규화에 추가 권장.
- **A3 강등 실제 이행(결정)**: `_run_script`에 `--epochs <계획값>` 주입 시 계획 이행 가능하나 GPU 시간 급증 + 휴리스틱 파싱 리스크 → 사용자 결정 사항으로 남김(현재는 감지만).
- **커밋**: A1/A2/A3 수정은 아직 미커밋(base/publish 양쪽). 커밋 준비 요청 시 publish 폴더 patch 방식으로 반영.

### 결론
S2 패널이 지목한 파이프라인 레벨 결함(검사기 오탐→파괴적 수리 / 수리 게이밍 / 실행-계약 불일치 은폐)을 **전문가 협업으로 근본 수정**하고 E2E 회귀 검증까지 완료. 이후 시나리오(S1/S7 등)는 이 수정의 혜택을 받는다.

## 후속 결정 필요 (범위 밖, 별도)
- **A3 강등 근원의 실제 수정**: `_run_script`에 `--epochs <계획값>`을 주입하면 계획 epoch을 실제로 이행. 단 (a)epoch↑=런타임 급증(GPU 시간), (b)success_criteria 텍스트 파싱이 휴리스틱이라 신중해야 함 → 지금은 **감지만** 넣고, 이행 주입은 사용자 결정으로 남김.
- 커밋: 이 3개 수정은 base/publish 어디에도 아직 미커밋.
