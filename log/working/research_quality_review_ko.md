# 연구 품질 비판 검토 회의록 (2026-07-30)

**질문(사용자):** 완주한 실험 결과가 "인간 연구원들이 모여 최선을 다해 낸 결과"라고 볼 수 있는가? 아니면 "한 번 끝까지 돌린 것"에 만족하는가?

**답(3전문가 만장일치): 아니요. 아직 인간 연구원 수준이 아니다.** 무결성(정직·비조작)은 확보됐으나, **성능·방법론·자율성 세 축 모두에서 "완주 증거"에 머물러 있다.** 특히 vision은 토이 수준.

참석: 성능/벤치마크 · 실험 엄밀성 · 자율연구 행동 비판.

---

## 1. 성능 판정 (문헌 기준 대비, 숫자는 아티팩트와 일치·조작 없음)

| 시나리오 | 실측 | 문헌 통상 | 판정 |
|---|---|---|---|
| S1 CIFAR-10 ResNet-18 | **35.67%** (1 epoch) | ~93–95% | ❌ **토이** (1/3 수준, 미수렴) |
| S1 MobileNetV2 | 23.88% | — | ❌ 토이 |
| S2 CIFAR-100 ResNet-50 | **16.0%**/top5 42.6% (1 epoch) | ~75–80% | ❌ **토이** (1/5 수준) |
| S2 ViT-tiny | 15.3%/41.9% | — | ❌ 토이 |
| S4 Titanic LR/RF | acc 0.83 / ROC 0.86 | 0.77–0.83 | 🟡 표준(교과서 sanity-check, SOTA 아님) |
| S5 California GB | R² **0.776** / RMSE 0.542 | 0.80–0.85 | 🟡 표준 하단 |

→ **vision은 "성능"이 아니라 "손실이 내려가기 시작했다"는 증거.** 미수렴 구간이라 ResNet vs ViT/MobileNet 우열 주장은 통계적 소음. tabular만 방어 가능(단 SOTA 아님).

## 2. 방법론 판정 (생성 코드 분석)

- **S1 CIFAR-10 = 최소 실행 코드**: 1 epoch(계획 3), batch 32(계획 128), **ImageNet용 아키텍처를 32×32에 미개조**(공정성 훼손), val split 없음(test-set peeking 구조), 튜닝·seed반복 없음.
- **S2 CIFAR-100 = 골격은 상위, 실행이 배신**: 224 리사이즈·top-5·optim 팩토리 등 코드는 연구원급이나 1 epoch·15%로 무의미. `train_eval.py`는 유령(죽은) 모듈.
- **S5 California = 설계는 연구원급, 미발현**: CV 하이퍼파라미터 튜닝·permutation importance·오차분석 코드 존재하나 **튜닝이 한 번도 실행 안 됨** → 리포트는 sklearn 기본값 1회. seed 단일.

  > **⚠️ 2026-08-04 원인 정정** — 차단 지점은 `tune=False` 플래그가 아니다 (`docs/expert_panel_2026_08_ko.md` §8.1)
  > 실제 원인은 **CLI 계약에 config 전달 채널이 없는 것**이다. 생성 코드 `experiment_impl.py:284`가 `exp_cfg = getattr(args, "experiment_config", None)`으로 읽는데 `cli.py:9-31`에 `--experiment-config` 같은 인자가 없어 **항상 `None`** → `lr_cfg`/`gbr_cfg`/`gbr_grid_cfg`/`perm_cfg` 전부 None → grid 가드 false → sklearn 기본값 경로. **`enabled=True`로 바꿔도 아무것도 달라지지 않는다.**
  > 증거: `dce015`의 R² `0.7756446042829697`가 무튜닝 앵커 `c0dad6ae71a8`과 **비트 동일**. 그리드의 `n_estimators` 후보(200/400/800)는 sklearn 기본 100과 겹치지 않으므로 그리드가 돌았다면 값이 같을 수 없다 → 확정적 미실행.
  > **그리고 이것은 S2 패널 결함 #3(`hasattr(cfg,"optim")` False → 선언한 SGD/lr0.1 무시하고 AdamW/3e-4 실행)과 동일한 단일 아키텍처 구멍이다.** 두 문서가 이를 서로 다른 두 버그로 기록해왔다.

**공통 결함 Top 3:**
1. **학습 예산 강등 (계획 3 epoch → 실제 1 epoch)** — 파이프라인이 `--epochs`를 실험 CLI에 전달 안 함(`phase3_execution.py` L312-315 주석이 자인). vision 성능을 통째로 망침.
2. **통계적 엄밀성 전무** — 세 실험 모두 단일 seed·단일 실행. 신뢰구간·분산 없음. S1의 Δ(+11.79%p)에 오차막대 없어 유의성 판단 불가.
3. **정교한 방법론 코드가 죽은 코드로 방치** — S5 CV 튜닝·S2 scheduler가 실행 경로에 배선 안 됨. "연구원처럼 보이는 껍데기"만 존재.

## 3. 구조 진단 — 왜 이런가 (핵심)

**시스템이 "성능 최대화"가 아니라 "정직한 1회 완주"를 목표로 설계됐다.**
- `run_execution_phase`는 **딱 1회 호출**(outer loop 없음). 성공 게이트 통과 즉시 `return success=True`.
- 모든 게이트(L2 실행성공 / L4 스텁·NaN / A3 계약)는 **무결성 검사** — 값이 *좋은지*(품질)는 **어디서도 안 봄**. top1=0.09든 0.95든 동일하게 성공.
- **planner의 `success_criteria`(성능 목표)가 실제 지표와 정량 비교되는 곳 0곳** — "top1≥70%"라 써도 기록만 하고 달성 시도 안 함.
- `max_experiments`(기본 3)는 **죽은 설정**(루프 바운드로 안 읽힘). Phase 3 `while`은 *고장 수리* 루프이지 *성능 개선* 루프가 아님. `EXECUTION_SCALE_DOWNGRADE`(3→1 epoch)는 감지만 하고 막지도 재실행하지도 않음. `ExtensionProposer`는 제안만.

## 4. 무결성 결함 (성능 이전 문제, 반드시 수정)

- **S1 논문 본문이 "3 epochs"를 반복 서술하나 실제 result.json은 `epochs:1`** — writer가 계획값(planned=3)을 실측 검증 없이 본문에 박음. **논문이 실제 실행과 불일치**(재현성·정직성 결함).

## 5. 개선 로드맵 (완주기 → 연구원으로)

**P0 (즉효·저비용):**
1. **`--epochs`/`--batch-size` 파이프라인 전달 버그 수정** — 계획된 학습 예산이 실제로 실행되게. (vision이 비로소 실제 학습)
2. **writer가 계획값이 아닌 실측값(actual epochs/metrics)을 보고**하도록 수정(무결성).

**P1 (구조적·"성능 추구"로 전환):**
3. **목표-지표 게이트**: `success_criteria`를 정량 파싱(예 "top1≥0.70")해 실제 primary metric과 비교 → `TARGET_MISSED` 판정. planner 스키마에 `primary_metric`+`target_value` 구조 필드 강제.
4. **오케스트레이터 개선 outer loop**: 죽은 `max_experiments`를 살려, `TARGET_MISSED`이면 analyzer가 개선 처방(epoch↑/LR/모델/증강) → coder 반영 → Phase 3 **재실행**을 목표 달성/예산 소진까지 반복. (실행 성공 이후에도 도는 루프)
5. **실험 이력·best-so-far 추적** + 수렴 판정(개선 없으면 조기 종료).

**P2 (엄밀성):**
6. **multi-seed 반복 + 신뢰구간** 보고, 죽은 튜닝 코드(S5 CV) 실행 경로 배선, vision 아키텍처 CIFAR 개조 + val split(누수 차단).

## 5-1. 구현 현황 (2026-07-30)
- ✅ **P0** (학습예산 버그·무결성): `--epochs` 실험 전달 + writer 실측 보고. CIFAR-10 ResNet-18 35.67%@1ep→79.36%@3ep, 논문 epoch 일치.
- ⚠️ **P1** (완주→성능추구): `target_gate`(목표 파싱·주지표 선택·met/missed) + orchestrator **개선 outer loop**(목표 달성 시 조기종료 / 없으면 정체까지 epoch×3 증대 / best-so-far / `max_experiments` 부활). **outer loop는 실전 검증됨**: S1에서 iter0(3ep, top1 62.99) → 예산 9ep 증대 → iter1(9ep, top1 68.86) → best 유지(events.jsonl 확인).

  > **⚠️ 2026-08-04 정정 — "목표 게이트"는 검증되지 않았다** (`docs/expert_panel_2026_08_ko.md` §4-④, §13)
  > - `target_gate.parse_targets`를 실제 3개 run의 `success_criteria`에 직접 실행한 결과 **전부 `[]`를 반환**했다. events.jsonl도 일치한다: `개선 루프 1/2: top1=62.9900 (no_target)`. **즉 실전에서 발동한 정책은 "epoch×3, 반복 상한까지"뿐이고 목표-지표 비교는 0회 수행됐다.**
  > - planner가 생성하는 `success_criteria`는 대부분 정성·재현성 문구("±0.5%p 이내 유지")여서 정량 목표가 존재하지 않는다.
  > - **`target_gate.py:131`에 스케일 버그**: `tgt = matched["value_frac"] if value <= 1.0 else matched["value"]` — 지표가 퍼센트(68.86)이고 목표가 분수 표기(`top1 >= 0.70`)면 `68.86 >= 0.7`로 비교해 **목표 70%를 68.86%에서 "달성"으로 선언하고 루프를 조기 종료**한다.
  > - **더 근본적 결함**: 루프의 결정 지표가 **test set**이다(워크스페이스에 `random_split`/`val_loader`/`Subset(` 0건). best-so-far 갱신과 epoch 증대 판단을 둘 다 test 정확도로 하므로 P1은 사실상 test set에 대한 적응적 하이퍼파라미터 탐색이다 → **리뷰어 즉사 사유**.
  > - iter1에서 멈춘 이유도 "정체"가 아니라 **반복 상한(`max_iters=2`) 소진**이다. 62.99→68.86은 +9.3% 상대 개선으로 `PLATEAU_EPS=0.02`를 4.6배 초과한 상태, 즉 **여전히 오르는 중에 끊겼다.**
  > - tabular(S4/S5)은 `base_epochs=None`이라 iter 0에서 즉시 break → **개선 루프의 혜택이 0**이다.
  >
  > → 대응: 사전등록 프로토콜에서 threshold를 planner 산문이 아닌 **task spec 파일**에서 받으므로 정규식 파싱 자체를 폐기한다. val split 도입은 Track 3-4~3-7.
- ⏳ **P2**(미착수): multi-seed+신뢰구간, 죽은 튜닝코드(S5 CV) 배선, vision 아키텍처 CIFAR 개조+val split, LLM 처방형 개선(epoch 외 LR/증강/모델).

## 6. 한 줄 결론
> 현재 MARS는 **"정직하게 한 번 완주하는 실행기"**다. 무결성 방어(스텁/NaN/버전)는 정교하나 **품질을 향한 반복 루프가 구조적으로 부재**하고, vision은 1-epoch 토이라 연구 결과로 낼 수 없다. "인간 연구원 수준"이 되려면 **①학습예산 버그 수정 → ②목표-지표 게이트 → ③개선 outer loop**가 필요하다.
