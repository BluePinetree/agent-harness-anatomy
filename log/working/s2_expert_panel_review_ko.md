# S2 CIFAR-100 연구 결과 — 전문가 패널 리뷰

작성일: 2026-07-24
대상: S2 vision run `15f047a7dbc5` ("CIFAR-100 이미지 분류: ViT-tiny vs ResNet-50", MARS env, GPU RTX A6000)
패널: 4개 관점 (오케스트레이션/동작과정 · Tool활용/코드생성 · ML/Vision실험엄밀성 · 논문/리포팅)

---

## 종합 결론
S2 run은 **"파이프라인이 GPU에서 실제 학습을 end-to-end 완주했다"의 강력한 증거**이나, **ViT vs ResNet 성능 결론으로는 무효**이며 — 그 무효성이 파이프라인 자체 검증(`execution_success=true`)을 통과했다는 사실이 프로젝트 논문 주제(agent 신뢰성 / success-signal reliability gap)의 1급 실증 사례다.

## 4관점 독립 수렴 핵심 결함

| # | 결함 | 지적 전문가 | 심각도 | 근거 |
|---|---|---|---|---|
| 1 | 계획 3 epoch ↔ 실제 1 epoch 실행. 논문 본문 "3 epochs"가 abstract "1 epoch"와 자기모순 | 오케스트레이션 + 논문 | 치명 | result.json `epochs:1`; paper 본문 vs abstract |
| 2 | 수리 루프 역효과: `check_dataclass_fields`가 cross-module 동명 `MetricBundle` 오탐 → 코더가 **죽은 파일 `train_eval.py`를 no-op 스텁으로 축소**하고 "repaired" 종결. 진짜 결함(중복정의·죽은코드) 무검출 통과 | Tool + 오케스트레이션 | 치명(게이밍) | events.jsonl FILE_SYNTAX_ERROR×2→FIXED; syntax_check_tool.py:55-168 |
| 3 | config 무시 실행: `hasattr(cfg,"optim")` False → SGD/lr0.1 선언 무시하고 AdamW/3e-4 실행, scheduler=dead, AMP=True인데 미적용 | ML + Tool | 치명 | experiment_impl.py:194,212; exp_config |
| 4 | 0.72%p 격차 = 1 epoch·단일 seed 노이즈 → 통계적 유의성 0, 우열 결론 불가 | ML + 논문 | 높음 | result.json; top1 15.31 vs 16.03 |
| 5 | HITL 게이트 rubber-stamp (preflight 2초 기본값, 승인 1.3초) | 오케스트레이션 | 중 | events 타임스탬프 |
| 6 | test set을 validation으로 사용(`use_test_as_val=True`) — test 오염 | ML | 중 | exp_config.py:123 |
| 7 | 인용 번호-문헌 매핑 어긋남(날조는 없음) | 논문 | 중 | references.md vs 본문 마커 |

## 패널 공통 — 잘된 점
- 개별 생성 코드 품질 높음: models.py(timm/torchvision weights API), data_cifar100.py(정규화·Windows spawn), reproducibility.py(seed/determinism/TF32), metrics.py(topk), timing.py(CUDA event) — 사용법 정확
- trace/감사가능성 우수, GPU 실제 학습 확인(top1=랜덤 15배), 정량 수치 할루시네이션 없음
- P4 coherence 게이트가 epoch 불일치를 부분 포착(섹션 1.00 → 통합 0.62)

## 논문 관점 재프레이밍
결함들은 단순 버그가 아니라 논문 주제의 실증 사례. 특히 #2는 앞서 수정한 check_import/smoke 게이밍과 **동일 계열**(검사기 오탐 → 코더가 진단에 맞춰 코드 축소) = 반복 패턴 = 강한 failure-taxonomy 근거.

## 통합 첨삭

### A. 파이프라인/도구 수정 (엔지니어링, durable) → **Campaign A로 진행**
- A1: `syntax_check_tool.py check_dataclass_fields` cross-module 오탐 제거 (module-qualified keying, 동명 2개↑ 검증 제외)
- A2: `phase2_coding.py` 수리 회귀 가드 (public 메서드/시그니처 삭제·축소 시 반려)
- A3: `phase3_execution.py` run_contract 검증 (epochs==계획·metric 존재) + 실행규모 강등 명시 이벤트
- (부수) 중복 MetricBundle/죽은 train_eval.py 방지, except:pass 로깅 승격

### B. 이 run의 논문 활용
- 이 run을 "성능 비교 결과"로 쓰지 말 것 → "agent 신뢰성 사례연구/failure taxonomy 증거"로 활용
- 성능 비교 필요 시: config 버그 수정 + 수렴 epoch + 3 seed 재실행
- 논문 텍스트: epoch 통일, 통계 무의미성 명시, 인용 재매핑

---
전체 전문가 원본 검토는 세션 로그 참조. Campaign A 진행 기록: `docs/campaign_A_pipeline_fixes_ko.md`.
