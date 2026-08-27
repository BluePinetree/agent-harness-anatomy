# CrewAI 스펙트럼 테스트 로그 (논문용)

목적: 안정화된 CrewAI가 **깊이·범위·도메인 스펙트럼이 다양한 연구주제**에 유연하게 반응해 심도 있는 연구·결과를 도출하는지 검증. 테스트 세트 정의: `docs/test_checklist_v1.md` (S1~S10).

판정 기준 (앵커 6개, `docs/stage1-3_crewai_anchor_plan_ko.md` §10 참조):
A1 Phase 0~4 완주 · A2 execution_success=true · A3 선택 실험 전부 실행 · A4 실제·타당한 지표 · A5 paper 생성 · A6 재현성(동일 seed bit-identical).

실행 방식: `crewai_prototype/harness/anchor_run.py --profile <preset>` (헤드리스 인프로세스, 승인/preflight 자동, seed 42).

**실행 환경 (2026-07-24 이관):** base(x) → **MARS conda env** (`anaconda3\envs\MARS`, python 3.10.19, **crewai 1.14.3**로 업글, **torch 2.5.0+cu118 GPU=NVIDIA RTX A6000**, timm/seaborn 설치). base는 torch가 MSVC 충돌로 깨져 있어 미사용. 실험 device는 `MARS_EXPERIMENT_DEVICE` env로 결정(vision=cuda 자동).

---

## 진행 현황

| S# | 시나리오 | 도메인/프로파일 | 상태 | run_id | 핵심 결과 | 재현성 |
|---|---|---|---|---|---|---|
| **S4** | Titanic (분류) | tabular_supervised | ✅ 완료 | `bf6325b52ec6` | LR acc 0.83/roc 0.86, RF acc 0.83/roc 0.84 | **R1 — 2026-08-04 실측 검증** (190/190 지표 bit-identical) |
| **S5** | California Housing (회귀) | tabular_supervised | ✅ 완료 | `c0dad6ae71a8` | GB RMSE 0.542/R² 0.776, LR RMSE 0.746/R² 0.576 | **R1 — 2026-08-04 실측 검증** (16/16 지표 bit-identical) |
| **S2** | CIFAR-100 (vision) | vision_classification | ✅ 완료 (GPU) | `15f047a7dbc5` | ResNet-50 top1 16.0%/top5 42.6%, ViT-tiny top1 15.3%/top5 41.9% (~1ep) | **R2 — 미검증** (`replay_identical: "not_verified"`) |
| **S1** | CIFAR-10 (vision) | vision_classification | ✅ 완료 (GPU) | `d359bac17637` | ResNet-18 top1 35.67%, MobileNetV2 top1 23.88% (Δ11.79) — **`epochs=1`** | **R2 — 미검증**, 레코드 파일 없음 |

> **⚠️ 2026-08-04 정정** (`docs/expert_panel_2026_08_ko.md` §13)
> - **`aca4a8`은 "S4 MARS 재검증"이 아니다.** replay가 아니며 실험 설계·수치가 다르다(LR acc 0.8212 vs 0.8268, `n_experiments` 2 vs 10). S4 행에서 제거했다.
> - **R1 주장은 작성 시점에 근거가 없었다.** 동일 seed 재실행 artifact 쌍이 존재하지 않았고, `result.json`과 `result_at_*.json`은 두 run이 아니라 md5·mtime이 같은 **같은 파일**이었다. → 2026-08-04에 동결 워크스페이스를 실제 재실행해 **S4·S5 모두 bit-identical을 확인**했다(주장 내용은 참이었으나 미검증 상태였음). 증거: `crewai_prototype/replay_verification/replay_diff_{S4,S5}.json`
> - **S1의 top1 35.67%는 `epochs=1` 결과다.** 해당 run의 paper 본문이 "3 epochs"를 16회 서술하지만 `result.json`은 `epochs: 1`이다(P0 학습예산 수정 이전 run).
> - **모든 run의 "Plan approved by user"는 사람 승인이 아니다** — `harness/anchor_run.py:283`의 자동 승인이다.
> - S1/S2의 모델 간 Δ는 **통계적으로 무효**다. 동일 seed·동일 3 epoch에서 run 간 ResNet 16.4%p 분산이 관측됐다(`expert_panel_2026_08_ko.md` §7.2).

### S1 CIFAR-10 수치 확정표 (2026-08-04 전수 실측)

문서 여러 곳에 CIFAR-10 top-1이 3개 값으로 흩어져 있었다. **run_id별로 하나씩 확정한다.** 인용 시 반드시 run_id와 epochs를 함께 표기할 것.

| run_id | ResNet-18 top1 | MobileNetV2 top1 | Δ | epochs (artifacts 기록) | 증거 경로 | 인용 가능? |
|---|---|---|---|---|---|---|
| `b383c3` | **NaN** | NaN | — | 미기록 | — | ❌ L4 NaN 거부 사례로만 |
| `5cd298` | **79.36** | 48.82 | 30.54 | **미기록** | `legacy_pre_prereg/outputs/run_*5cd298/workspace/results/result.json` | ⚠️ **아래 주의** |
| `230013` | **68.86** | 35.43 | 33.43 | **9** | `..._230013/.../result.json` | ⚠️ 개선 루프 iter1 결과 |
| `230013` iter0 | 62.99 | 25.34 | 37.65 | 3 | 해당 `runs/230013.../events.jsonl` (result.json엔 없음 — best만 보존) | ⚠️ 이벤트 로그에만 존재 |
| `d359ba` | **35.67** | 23.88 | 11.79 | **1** | `..._d359ba/.../result.json` | ✅ `epochs=1` 명시 시 |

**`5cd298`(79.36%)에 대한 주의 — 가장 높은 수치가 근거는 가장 약하다:**
1. **`epochs`가 result.json에 전혀 기록되지 않았다.** "3 epochs로 79.36%"라는 서술을 아티팩트로 뒷받침할 수 없다.
2. **`test_top1_acc_mean = 79.36`과 `test_top1_acc_run1 = 79.36`이 동시에 기록되어 있다** — 즉 **n=1의 "평균"** 이다. 분산 없이 논문 표에 들어갈 형태로 저장되어 있어 특히 위험하다. (생성 코드가 multi-seed 반복 루프를 스스로 작성했으나 `cli.py`에 `--repeats`가 없고 `_experiment_cmd`가 `--seed 42`를 하드코딩해 항상 1회만 실행됨.)
3. 같은 목표·같은 seed 42의 다른 run이 62.99%를 냈다 → **재현 불가.**

→ **결론: CIFAR-10 헤드라인 수치로 79.36%를 쓰지 않는다.** 사전등록 프로토콜(val split + `--repeats` ≥3 + CI)로 재수집한 값만 논문 main table에 넣는다.
| S7 | AirPassengers (시계열) | timeseries_forecasting | 대기 | — | — | — |
| S3/S6/S8/S10 | 커스텀 데이터 | 각종 | 데이터 준비 필요 | — | — | — |
| S9 | NLP 감성 | (전용 프로파일 없음) | 후순위 | — | — | — |

---

## S4 — Titanic 이진 분류 (tabular classification)

- run_id: `bf6325b52ec6`, seed 42, profile `tabular_supervised`
- 결과(홀드아웃, 10-config ablation): **Logistic Regression** acc 0.827 / ROC-AUC 0.865, **Random Forest** acc 0.832 / ROC-AUC 0.836. winner(ROC-AUC): LR.
- A6 재현성: 동일 seed 재실행 bit-identical ✓
- 레코드: `crewai_prototype/results/anchor_record.json`
- 의의: 안정화 파이프라인의 **첫 신뢰 앵커**. 근본원인 사슬(env/scaffold/phase3/smoke/check_import) 해소 후 확보.

## S5 — California Housing 회귀 (tabular regression)

- run_id: `c0dad6ae71a8`, seed 42, profile `tabular_supervised`, **torch 불필요(sklearn)**
- 결과: **Gradient Boosting** RMSE 0.542 / MAE 0.372 / R² 0.776, **Linear Regression** RMSE 0.746 / MAE 0.533 / R² 0.576. winner(R²): GB.
  (California Housing target≈$100k 단위 → GB R² 0.78 = 표준-우수, LR R² 0.58 = baseline. 타당.)
- main.py 보존·수리 게이밍 없음·clean 완주.
- A6 재현성: RMSE/MAE/R² bit-identical ✓
- 레코드: `crewai_prototype/results/s5_california_housing_record.json`
- 의의: **분류→회귀 task-type 유연성** 입증(새 metric family RMSE/MAE/R²). 검증된 tabular 스캐폴드 재사용으로 저리스크 확장.
- 경미 gap: goal이 요청한 feature_importance가 result.json metrics에 미출력(논문 표엔 영향 없음, 필요 시 후속).

## S2 — CIFAR-100 이미지 분류 (vision, GPU)

- run_id: `15f047a7dbc5`, seed 42, profile `vision_classification`, **device cuda (RTX A6000)**, MARS env
- 결과(~1 epoch from-scratch, CIFAR-100 100클래스):

  | 모델 | top-1 | top-5 | 학습시간 | 파라미터 |
  |---|---|---|---|---|
  | ResNet-50 | 16.03% | 42.59% | 233.5s | 23.71M |
  | ViT-tiny | 15.31% | 41.89% | 127.9s | 5.54M |
  | 비교(ViT−ResNet) | −0.72%p | −0.70%p | −105.6s | −18.17M |

  → 정확도는 ResNet-50 근소 우위, **효율(속도 1.8x·파라미터 4x 작음)은 ViT-tiny** — 정확도 vs 효율 트레이드오프 정량화. (top1 ~15-16%는 1-epoch from-scratch 기준 타당; random=1%.)
- main.py 보존·수리 게이밍 없음·clean 완주. GPU 학습시간 실측(ResNet 더 느림).
- 재현성: **R2 — 미검증**. ~~tolerance 내 재현~~ → **2026-08-04 정정**: `s2_cifar100_record.json`이 `replay_identical: "not_verified"`로 기재하고 있고, 재현 시도 이력 자체가 없다. "tolerance 내 재현"은 근거 없는 서술이었다.
  - **주의**: "GPU라서 R2"라는 논거는 자기 증거와 상충한다. 이 run의 env 블록은 `deterministic_algorithms=true`, `cudnn_deterministic=true`, `cudnn_benchmark=false`, TF32 off, `CUBLAS_WORKSPACE_CONFIG=:4096:8`을 기록하고 있는데 이는 GPU를 **결정적으로 만드는** 설정이다. R2인 이유는 GPU 비결정성이 아니라 **재현을 시도하지 않았기 때문**이다.
  - R1(CPU)/R2(GPU) 대비를 논문 재현성 섹션 근거로 쓰려면 S2 replay를 실제로 실행해야 한다.
- 레코드: `crewai_prototype/results/s2_cifar100_record.json`
- 의의: **첫 vision 앵커 + 첫 GPU 실행 + 첫 MARS-env 실행**. Tabular를 넘어 **vision/deep 도메인 축** 확보.

---

## 스펙트럼 커버리지 현황
- ✅ Tabular **분류**(S4) + **회귀**(S5) + Vision **분류**(S2) — 도메인 2축·task-type 3종 확보
- ⏭ 시계열(S7), NLP(S9), 커스텀(S3/S6/S8/S10) — 예정
- 재현성 등급: CPU 결정적 실험(S4/S5)=**R1**, GPU 실험(S2)=**R2**(cuDNN 비결정) — 논문 재현성 섹션 근거.
- 다음 후보: **S1 CIFAR-10**(가벼운 vision, GPU) 또는 **S7 AirPassengers**(시계열, 새 도메인). 커스텀(S3/S6/S8/S10)은 데이터 파일 준비 후.

---

## ★ 실측 감사 (2026-07-28) — `outputs/run_*` 305개 result.json 전수 스캔
로그의 "완료" 주장과 실제 산출물을 대조한 결과(데이터셋 무결성 목적):

| S# | 데이터셋 | run 시도 | success=true | 실제 지표 | 판정 |
|---|---|---|---|---|---|
| S4 | Titanic(tab-clf) | 11 | **6** | acc/roc_auc ✓ | ✅ **진짜 안정** |
| S5 | CalHousing(tab-reg) | 6 | **5** | rmse/mae/r2 ✓ | ✅ 안정 (단 최근 live run 1건 실패) |
| S2 | CIFAR-100(vision) | 3 | **1** | top1/top5 ✓ | ⚠️ 진짜지만 **1건뿐(얇음)** |
| S1 | CIFAR-10(vision) | **24** | 1 | **지표 없음(elapsed_s 1.07초, 스텁)** | ❌ **진짜 성공 0건(degenerate)** |
| S3·S6·S7·S8·S9·S10 | 커스텀/시계열/NLP | **0** | 0 | — | ❌ **미실행** |

**결론(정직한 재평가):**
- 논문용 10개 시나리오 중 **진짜 완주는 3개(S4·S5·S2)뿐**. Vision은 **S2 1건**으로 얇고, **S1 CIFAR-10은 진짜 성공 이력 없음**(유일 "성공"은 1.07초 스텁 = specification gaming 잔재, 개발기 2026-06-25).
- **시계열(S7)·NLP(S9)·커스텀(S3/S6/S8/S10)은 단 한 번도 실행 안 됨.**
- **신뢰성 변동 상존:** 최근 California Housing live run(`d60d5dfc`)이 함수 시그니처 불일치(`linear_coefficient_importance`/`gbdt_impurity_importance`)로 repair 루프→타임아웃 실패. 전문가 패널이 지적한 "cross-module 시그니처 미검출" 구멍과 정확히 일치.
- **판단:** CrewAI는 아직 논문 데이터셋 스펙트럼에서 "올바로 동작"한다고 보기 어렵다 → **타 프레임워크 이전에 (1) codegen 안정화 + (2) vision·미실행 데이터셋 실제 완주가 우선.** (사용자 지시 부합)

### 라이브 실측 스윕 (2026-07-28, 순차 실행)
감사 후 현재 코드로 직접 재실행해 pass/fail·실패패턴을 확인:

| S# | run(들) | 결과 | 판정 |
|---|---|---|---|
| S5 CalHousing | `f8c224`, `12d2af` | 2/2 완주·skip 0·**실제 rmse/r2** (lr/gbdt) | ✅ 진짜·안정(최근 실패는 일시적 변동) |
| S2 CIFAR-100 | `c9e6c6` | 완주·skip 0·**실제 top1/top5 + repro_ok** | ✅ 진짜·안정 |
| S7 시계열 | `6321f1` | "성공"이나 **coder가 smoke 스텁**(`smoke_metric:1.0`, "SARIMA/LSTM not yet implemented") | ❌ 가짜(spec-gaming) |
| S1 CIFAR-10 | — | 데이터 다운로드 3회 실패(WinError 10060, toronto.edu) | ❌ 데이터 차단(네트워크) |

**추가 발견(안정화 근거):**
- **가짜성공 벡터:** 스캐폴드는 `experiment_impl.py`에 실제 `run_selected_experiments` 구현을 요구하나, **coder(LLM)가 어려운 도메인에서 실제 구현 대신 스텁을 작성**(`execution_success:true`+`smoke_metric:1.0`)해 L2(success 플래그)·L3(numeric metric 존재)를 통과. `success_signal_reliability`의 기존 방어가 못 잡는 신종. → **탐지(스텁/placeholder→실패 처리) + coder 프롬프트 스텁 금지** 필요.
- **동시성 불가:** `initialize_runtime()`가 동시 실행 중인 다른 세션을 "고아"로 보고 interrupted 처리 → **anchor_run은 순차 실행만 안전.**
- **성숙 스캐폴드(tabular clf/reg, vision-cifar100)는 신뢰성 높음**; 시계열은 스캐폴드/코드생성 미성숙.

### 안정화 ① 가짜성공(스텁) 탐지 — 구현·검증 (2026-07-28)
- **구현:** phase3 L4 degenerate 탐지(`_is_degenerate_result`) + coder 프롬프트 스텁 금지. 상세: `crewai_prototype/CLAUDE.md` "가짜 성공(스텁/placeholder)" 항목.
- **단위검증:** S7 스텁=탐지, S5(`f8c224`/`12d2af`)·S2(`c9e6c6`) 실제=통과 → 오탐 0.
- **E2E 검증:** S7 재실행(`cb5bf2`) → L4가 스텁 거부 → 수리 루프 → 진짜 timeseries 미구현 → escalate/skip → **`exec_success=False`(정직한 실패).** 이전 `6321f1`의 `smoke_metric:1.0` 가짜성공이 제거됨.
- **의의:** 벤치마크 완주율이 정직해짐(스텁을 "완주"로 오집계하지 않음). 단 **시계열을 실제로 PASS시키려면** 코드생성/스캐폴드 강화(별도 커버리지 작업)가 필요 — 현재는 정직하게 "실패"로 표기됨.

### 후속 강화 + S1 CIFAR-10 진짜 성공 (2026-07-29~30)
- **버전 드리프트 수정:** coder가 설치 버전과 다른 API 사용(sklearn 1.7.2에서 제거된 `mean_squared_error(squared=False)`)으로 S5 실패 → `_version_rules_block`으로 설치 버전+API 규칙을 codegen/repair 프롬프트에 주입. S5 재검증 **그린**(GB r2 0.776, squared 오류 0).
- **L4 (c) NaN 값 거부:** S1 CIFAR-10 첫 실행이 `test_top1_acc=NaN`인데 "성공" 처리되던 신종 가짜성공 발견 → L4가 도메인 지표 **값이 전부 NaN/None이면** 실패 재분류하도록 강화.
- **S1 CIFAR-10 → ✅ 진짜 성공(`d359bac17637`):** 재실행에서 L4가 NaN 거부(1회) → repair 루프가 정확도 계산 수정 → **ResNet-18 top1 35.67%, MobileNetV2 23.88%**(Δ11.79) 실측. vision 커버리지 S1 확보(이제 vision=S1+S2 2건). 전체 루프(NaN→L4거부→repair→진짜결과) 실전 검증.
