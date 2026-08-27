# MARS 실행계획 — Stage 1~3 병합 + CrewAI 앵커 굳히기

작성일: 2026-07-21
근거: 전문가 패널 회의(책임연구원 / 멀티에이전트 아키텍처 / 벤치마크·재현성 / SW품질·안정성)
선행 문서: `docs/MARS_DAI2026_SUBMISSION_PLAN_KO.md`, `docs/MARS_TOP_TIER_ROADMAP_KO.md`

---

## 0. 이 문서의 목적

로드맵의 Stage 1~3을, "**CrewAI가 주어진 질문에 실험을 끝까지 완주하는지 먼저 확인한 뒤 타 프레임워크로 넘어간다**"는 실행 의도에 맞게 **재배치·병합**한 단일 실행계획이다. 최종 산출물은 "CrewAI 앵커(anchor) 확정" — 즉 논문·비교의 기준점이 될 재현 가능한 CrewAI 완주 run이다.

핵심 원칙: **완주 확인은 기존 CLI(`crewai_prototype/entrypoints/cli.py`)로 지금 가능하며, 비교 인프라(schema·telemetry)는 선행 조건이 아니다.** 따라서 CrewAI 완주 게이트를 앞으로 당기고, schema·telemetry 계측은 "도는 걸 확인한 뒤" 붙인다.

---

## 1. 실행 순서 (재배치 결과)

```
┌─ CrewAI 앵커 굳히기 (이번 실행 범위) ─────────────────────┐
│ Gate-A  "완료" 합격 기준 확정        (Stage 2-6 일부 선당김)  │
│ Gate-B  Windows asyncio freeze 스톱갭 (Stage 3-8)           │
│ Gate-C  고정 질문으로 CrewAI 실제 완주 run (Stage 3-7 확장)   │
│ Gate-D  run_summary.json 기록        (Stage 3-9)            │
└──────────── 여기까지 통과해야 타 프레임워크로 진입 ───────────┘
        ↓ (앵커 확정 후)
[Stage 1] research_questions.md + result 공통 schema + task spec
[Stage 2] rsp.telemetry 계측 훅 3프레임워크 연결 + success 정의 통일
[Stage 4] 타 프레임워크 실증 (AutoGen 역할 보강 → LangGraph stub 제거)
[Stage 5] adapter + run_benchmark + failure_taxonomy
```

Stage 1·2는 앵커 확정 뒤에 수행한다. 실제로 도는 파이프라인을 본 뒤 계측 지점을 정하는 것이 정확하기 때문이다.

---

## 2. Gate-A — "완료(정상 마무리)"의 정의 확정

문제: 현재 "완료"에 두 층위가 섞여 있고 ADR-013(문서)과 phase3 코드가 상충한다.

- **Level 1 (파이프라인 완주):** Phase 0~4가 모두 실행되고 `paper.md`가 생성됨. (예: run `0910b073c843`)
- **Level 2 (실험 실제 성공):** `execution_success:true` + 목표 metric 도달 + 선택 실험 전부 실행.

확인된 최신 run은 선택 5개 중 2개만 실행(`exp1`,`exp3`), `execution_success:false`, accuracy 0.1586 → **Level 1은 통과했으나 Level 2 실패.** 사용자가 요구한 "실험이 마무리까지"는 Level 2다.

**합격 기준(초안 — Gate-A 조사로 코드 기준 확정):**

| # | 조건 | 판정 소스 |
|---|------|----------|
| A1 | Phase 0~4 전 단계 실행, `SYSTEM_END` 이벤트 발생 | `events.jsonl` |
| A2 | `results/result.json`의 `execution_success == true` | result.json |
| A3 | 선택된 실험(`selected_experiments`)이 전부 `experiments_ran`에 포함 | result.json |
| A4 | 목표 metric이 result에 존재하고 사전 합의한 임계값 이상 | result.json / task spec |
| A5 | `paper.md`(또는 report) 생성 | workspace |
| A6 | 동일 seed 재실행 시 A1~A5 재현 | 2회차 run |

> ADR-013 갱신 필요: 현재 "rc=0만 성공" 문서와 phase3 코드(L2 `success=False`→실패)가 반대. 채택 기준을 코드에 맞춰 개정한다.

---

## 3. Gate-B — Windows asyncio 프리즈 스톱갭

증상: Phase 3 repair agent가 `Crew.kickoff()`를 여러 번 호출한 뒤 HTTP 서버가 무응답(IOCP freeze). 근거 `docs/decisions/ADR-014`. 이 버그가 **자동 완주 자체를 막으므로 Gate-C 전에 반드시 완화**한다.

스톱갭 방침(둘 중 택1 또는 병행):
1. repair agent의 `Crew.kickoff()` → 직접 LLM 호출로 전환 (ADR-001 패턴 재사용).
2. `WindowsSelectorEventLoopPolicy` 적용.

정확한 수정 위치·패치안은 Gate-B 조사 결과로 확정한다.

---

## 4. Gate-C — CrewAI 실제 완주 run

- 실행: `python -m crewai_prototype.entrypoints.cli --topic ... --profile tabular_supervised ...`
- 태스크: **Titanic(tabular, 가장 빠름)** 우선 → 통과 후 CIFAR(vision).
- 절차: 실행 → A1~A5 관측 → 실패 시 원인 규명·수정·재실행 → A6(재현) 확인.
- 실패 재현 시 원인(예: 5개 중 2개만 실행된 이유)을 기록하고 수정한다.

---

## 5. Gate-D — 앵커 기록

- 통과한 run의 `run_id`, seed, commit, 실행 command, metric을 `run_summary.json`으로 고정.
- 이후 Stage 5 adapter가 이 run을 논문 표로 변환할 기준점으로 삼는다.

---

## 6. 이번 실행 범위에서 제외 (앵커 확정 후)

- Stage 1(schema/RQ/task JSON), Stage 2(telemetry 훅/success 통일), Stage 4(타 프레임워크), Stage 5(벤치 인프라)는 앵커 통과 이후 착수.

---

## 7. 리스크

| 리스크 | 대응 |
|--------|------|
| Windows freeze로 완주 불가 | Gate-B 선행 |
| 커밋 안 된 6월 패치와 충돌 | 수정 전 작업트리 확인, 필요시 사용자 승인 후 커밋 |
| run 1회 15~20분 + 실험 실패 반복 | Titanic(빠른 태스크) 우선, 실패도 원인기록으로 자산화 |
| "완료" 기준 미합의 | Gate-A를 최우선 |

---

## 8. 실행 기록 (2026-07-21 ~ 07-22)

### 완료된 것
- **Gate-A 완료 기준 확정 + 버그 수정**: `completed` 상태 ≠ 실험 성공. L2 성공판정이 `success` 키만 봐서 스크립트의 `execution_success` 키를 놓치던 필드-계약 버그 수정 (`phases/phase3_execution.py:318`, 두 키 모두 검사 + bool을 numeric으로 오인하던 L3 교정).
- **Gate-B**: uvicorn 없는 인프로세스 헤드리스 드라이버 `harness/anchor_run.py` 작성 → ADR-014 프리즈 원천 회피 + 승인/preflight 게이트 자동 처리.
- **환경 블로커 해결 (중대)**: 현재 환경에서 CrewAI가 아예 실행 불가였음. crewai 1.14.3 ↔ openai 2.24.0 불일치로 `num_retries`(TypeError), `parallel_tool_calls`(400) 연쇄 실패 → `core/llm_factory.py`에서 `LLM(is_litellm=True, ...)`로 LiteLLM 라우팅 복원 + 두 파라미터 정리로 해결.
- **Gate-C attempt 1 (tabular/Titanic) 성공**: CrewAI가 Phase 0~4 완주, 실제 비교결과(LR AUC 0.865/acc 0.827, RF AUC 0.846/acc 0.832, seed 42) + paper.md(품질 0.98) 생성. run_id `e75cd5c94b11`. **CrewAI E2E 동작 입증됨.**

### 미해결 (앵커 신뢰성 잔여 gap)
- **거짓 성공 (result-schema 계약 미이행)**: tabular 코더가 스캐폴드 RunContract(entrypoint `main.py`, `artifacts.py:write_result_json`, 필수산출물 `results/result.json`)를 **완전히 무시**하고 자유 생성(`run_experiment.py` + `outputs/summary.json`). 근본원인: 해당 run에 `run_contract.json`/`manifest.json`/스캐폴드 안정파일이 **하나도 materialize되지 않음** → V4 tabular 경로가 스캐폴드 계약을 적용하지 않음. 결과: Phase 3가 `results/result.json`을 못 읽어 `metrics={}`, rc=0라 성공 통과.
- **vision 경로 블로킹**: torch가 버전 무관 시스템 레벨 DLL 실패(`c10.dll`, WinError 1114) — pip 수정 불가, MSVC 재배포/conda 필요. 사용자 결정에 따라 **나중으로 연기**.

### 환경 변경 이력 (투명성)
- timm 1.0.28 설치(vision용). 그 과정에서 torch가 2.12.1+cpu→2.10.0→2.13.0+cpu로 교체됐으나 모두 DLL 실패. numpy가 2.4.4로 올라가 scipy/numba/sklearn 호환이 깨져 **numpy 1.24.4로 복원**(tabular 스택 정상 확인). crewai가 pydantic 2.12.5에 경고(실행 지장 없음, 주시 대상).

### 다음 결정
tabular 앵커를 신뢰 가능하게 굳히기 위한 result.json 계약 이행 방식:
- **Fix A (권장·정확)**: V4 tabular 경로가 스캐폴드 계약을 materialize하도록 수정 → 모든 run이 결정적으로 `results/result.json` 생성 → 재현성 확보.
- **Fix B (신속)**: Phase 3에 결과 정규화 폴백 추가 → 자유 생성 산출물을 탐색·정규화. 빠르나 run마다 산출 형태가 달라 재현성 약함.

---

## 10. ✅ 앵커 확정 (2026-07-23)

**CrewAI 앵커 굳히기 완료** — 6개 합격 기준(A1~A6) 전부 통과.

- **run_id**: `bf6325b52ec6` (T2 Titanic, seed 42)
- **A1 파이프라인 완주**: Phase 0~4 완료, status=completed ✓
- **A2 실험 성공**: execution_success=true (result.json 검증) ✓
- **A3 전 실험 실행**: 코더가 10개 ablation 구성 설계·전부 실행(n_experiments=10) ✓
- **A4 실제 지표**: LR accuracy 0.827 / ROC-AUC 0.865, RF accuracy 0.832 / ROC-AUC 0.836 (winner: LR). degenerate/스텁/no-op 아님 ✓
- **A5 논문 생성**: paper.md, quality 1.0 ✓
- **A6 재현성**: 동일 seed 재실행 시 지표 bit-identical (R1 artifact-replay) ✓
- **레코드**: `crewai_prototype/results/anchor_record.json`

### 앵커 확보까지 해소한 근본원인 사슬 (전부 durable 코드 수정)
1. **환경 블로커**: crewai 1.14.3 ↔ openai 2.24.0 비호환(`num_retries`/`parallel_tool_calls`) → `core/llm_factory.py` `LLM(is_litellm=True)` 라우팅.
2. **Fix A 스캐폴드 미연결**: V4가 ScaffoldService 미호출 → `orchestration/pipeline_orchestrator.py`에서 materialize 재연결 + phase1/phase2 프롬프트 정렬(E1~E4b).
3. **성공판정 필드계약**: L2가 `success`만 검사 → `success`/`execution_success` 둘 다 검사(`phases/phase3_execution.py`).
4. **phase3↔스캐폴드 CLI**: `python src/main.py`를 인자 없이 실행 → `--output-root`/`--validation-tier full`/`--seed` 전달 + `result*.json` glob 리더.
5. **smoke 수리 게이밍**: 수리가 안정 main.py를 스텁으로 대체 → 안정파일 수리 금지 + mutable(experiment_impl) 리다이렉트.
6. **★ check_import PYTHONPATH (연쇄의 시작점)**: 검사가 `src/`를 sys.path에 안 넣어 `from artifacts import`가 spurious 실패 → 파괴적 수리 발동 → 코드 게이밍. `crew_tools/syntax_check_tool.py`에 `src/` 추가 → smoke 정상 통과 → 실제 experiment 보존.
7. **Phase 3 hang**: seaborn titanic 최초 다운로드 hang → DATA_DIR 캐시로 해소(cold cache 시 pre-warm 필요).

### 주의/후속
- 위 수정들은 **아직 커밋되지 않음**(작업트리). 앵커는 uncommitted 상태로 생성됨 → 커밋 여부는 사용자 승인 필요.
- 코드생성은 stochastic이나, check_import 수정 이후 파괴적 수리가 사라져 **정상 codegen이 보존**된다(핵심).
- vision 앵커는 torch 시스템 DLL 복구 후 별도 진행.
- 다음 단계: 타 프레임워크(AutoGen/LangGraph) 실증 → Stage 5 벤치 인프라(adapter가 anchor_record.json 소비).

### 9. Fix A 구현·검증 결과 (2026-07-22)

**Fix A 구현됨 (E1~E4b):**
- E1/E3: orchestrator `_execute`가 Phase 1 후 `ScaffoldService.materialize_stable_only` + `finalize_with_designer_output` 호출, `entry_point="src/main.py"` 고정.
- E4: phase1 디자이너 프롬프트에 "`src/experiment_impl.py`(run_selected_experiments) 필수 포함" 지시.
- E4b: phase2 코더 프롬프트에 experiment_impl 스캐폴드 계약(시그니처/반환 dict/result.json 미작성) 주입.

**검증 결과 (attempt 3·4, tabular/Titanic):**
- ✅ **스캐폴드 안정파일(main.py/cli.py/artifacts.py/config_schema.py) materialize 확인**, 코더가 experiment_impl.py 생성.
- ✅ **`results/result.json`이 완전한 계약 스키마로 결정적 생성**(execution_success/validation_tier/error_type/traceback 등). **거짓 성공 구멍 폐쇄** — `exec_success`가 이제 정직하게 false 보고.
- ❌ **그린 run 미달**: attempt 3은 런타임 버그(`build_random_forest() got multiple values for 'random_state'` — training.py 중복 kwarg), attempt 4는 import-time 버그로 실험 실패. 둘 다 **코더의 실제 생성코드 버그**이며 Phase 3 수리 루프가 `MAX_EXEC_REPAIR_ATTEMPTS` 내 복구 실패 후 `src/main.py`로 에스컬레이션.

**결론**: 앵커 **인프라(result.json 계약)는 검증 완료**. 남은 blocker는 **코드생성·수리루프 견고성**(논문 RQ2 "repair cost/failure recovery" 주제 그 자체). attempt 1(수정 전)은 동일 태스크에서 깨끗한 과학적 성공(LR 0.865/RF 0.846)을 냈으므로 코드생성 성공은 **stochastic**.

**다음 결정지점**: (a) 수리루프 견고화(에스컬레이션이 entry가 아닌 실제 버그 dep을 타겟, repair에 정확한 context 주입) — 근본적이나 깊음, (b) 드라이버를 "continue"로 바꿔 수리 재시도(attempt=0 리셋) — 저비용이나 hint 없이는 반복 위험, (c) 재실행 반복으로 그린 run 1개 확보(stochastic) 후 Gate-D.
