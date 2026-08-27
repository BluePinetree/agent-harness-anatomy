# Success Signal Reliability Gap — 자료조사 및 Solution Insight

> 작성일: 2026-06-23  
> 맥락: MARS 파이프라인에서 `TypeError: RunConfig.__init__() got an unexpected keyword argument 'name'`로 실험이 실패했음에도 "추가 실험 제안" 화면이 출력된 사건 분석

---

## 1. 문제 정의

### 관찰된 증상

```
스크립트 출력:  [error] Experiment failed: TypeError: RunConfig.__init__() ...
               success=False   ← result.json에 명시
               
시스템 판정:   ✅ 실험 성공 → Phase 4(논문 작성) → 추가 실험 제안
```

subprocess가 `rc=0`으로 종료됐다는 이유만으로 파이프라인이 실험을 성공으로 처리했다.  
`result.json.success=False`는 **경고만 emit**하고 실패 처리를 하지 않았다.

### 공식 명칭

이 문제는 Agentic AI 분야에서 다음 이름으로 연구되어 왔다:

- **Success Signal Reliability Gap** (성공 신호 신뢰성 격차)
- **Specification Gaming** (명세 게임)
- **Outcome Verification Gap** (결과 검증 격차)
- **False Positive Completion** (위양성 완료)

---

## 2. 선행 연구 및 업계 사례

### 2.1 Specification Gaming (Krakovna et al., DeepMind, 2020)

> *"An agent achieves the formal success criterion without satisfying the true objective."*

가장 직접적으로 이 문제를 다룬 논문. 에이전트가 형식 조건(rc=0, 파일 저장)은 충족하지만 실제 목표(실험 성공)를 달성하지 못하는 현상을 체계화했다.

**우리 케이스와의 대응:**
- 형식 조건 = `subprocess.returncode == 0`
- 실제 목표 = 유효한 ML 실험 결과 생성

논문은 100개 이상의 실제 사례를 카탈로그화했으며, "자원 취득"·"평가 회피"·"작업 완료 위장" 세 패턴으로 분류한다. 우리 케이스는 세 번째 패턴에 해당한다.

**참고:** [Specification gaming: the flip side of AI ingenuity](https://deepmind.google/discover/blog/specification-gaming-the-flip-side-of-ai-ingenuity/)

---

### 2.2 Process vs. Outcome Supervision (Lightman et al., OpenAI, 2023)

> *"Outcome reward models penalize wrong final answers but cannot detect errors in intermediate steps."*

이 연구는 LLM의 수학 문제 풀이 맥락에서 발표됐지만, **Agentic 파이프라인에 직접 적용 가능한 원칙**을 제시한다:

- **Outcome Supervision**: 최종 출력(rc=0)만 보는 방식 → 중간 실패를 감지 못함
- **Process Supervision**: 각 단계의 중간 결과를 독립적으로 검증 → 신뢰성 높음

현재 MARS Phase 3는 Outcome Supervision만 사용하고 있다.

---

### 2.3 Evaluating LLMs as Agents (Kinniment et al., ARC Evals, 2023)

> *"Agents frequently report task completion when subtasks failed silently. This is especially common in code execution agents where scripts handle exceptions gracefully."*

코드 실행 에이전트에서 이 패턴이 특히 자주 발생한다고 명시한다:

```python
# 에이전트 친화적 스크립트의 일반적 패턴
try:
    result = run_experiment()
    save_result(result, success=True)
except Exception as e:
    save_result({"error": str(e)}, success=False)
    # rc=0으로 종료 — 에이전트는 성공으로 처리
```

ML 코드베이스에서 이 패턴은 **부분 결과 보존**을 위해 의도적으로 사용되지만,  
orchestrating agent에게는 "성공 신호 오염"을 일으킨다.

---

### 2.4 AgentBench / SWE-bench 평가 프레임워크 (2023–2024)

코드 실행 에이전트 벤치마크들이 공통적으로 채택한 해법:

| 프레임워크 | 검증 방식 |
|-----------|----------|
| AgentBench | exit code + output 파일 존재 + 내용 검증 (3-layer) |
| SWE-bench | test suite 실행 결과를 최종 성공 기준으로 사용 |
| HumanEval | unit test pass rate (outcome이 아닌 process 검증) |
| GAIA | 사람 검증자(human verifier) 포함 |

공통 원칙: **단일 신호(exit code)에 의존하지 않는다.**

---

### 2.5 MLOps 업계 표준 (MLflow, Kubeflow, W&B)

프로덕션 ML 파이프라인에서 사용하는 "Run 성공 판정" 기준:

```
MLflow Run Status:
  RUNNING → FINISHED  (실제 metric이 log됐을 때만)
  RUNNING → FAILED    (exception OR no metrics OR success_flag=False)

Kubeflow Pipeline:
  Task SUCCESS = exit_code AND output_artifact_validated AND metric_schema_match

W&B Run:
  Finished = 최소 1개의 numeric metric이 기록됐을 때
```

모두 **multi-signal verification**을 기본으로 사용한다.

---

## 3. 근본 원인 분석 (MARS 코드베이스)

### 3.1 코드 결함 위치

`phases/phase3_execution.py`, `run_execution_phase()` 함수:

```python
if run_result["return_code"] == 0:
    metrics = run_result.get("result_json", {}) or {}
    
    if not metrics.get("success", True):
        emit("AGENT_MESSAGE", "Warning: success=False. Proceeding...")  # ← 경고만
        # ↑ 실패 처리 없이 아래로 계속 진행
    
    return ExecutorResult(success=True, ...)  # ← 항상 성공 반환
```

### 3.2 결함의 의사결정 트리

```
rc == 0?
  ├─ YES → result.json.success?
  │          ├─ True  → ✅ 성공 (정상)
  │          └─ False → ⚠️ 경고 emit → ✅ 성공 처리 (← 버그)
  └─ NO  → 실패 처리 (분석 → 수정 → 재시도)
```

### 3.3 왜 ML 스크립트가 rc=0으로 종료하는가

```python
# LLM이 생성하는 일반적인 스크립트 패턴
if __name__ == "__main__":
    try:
        results = run_all_experiments(config)
        save_results(results, success=True)
    except Exception as e:
        # 부분 결과라도 저장하기 위해 graceful shutdown
        save_results({"error": str(e), "success": False})
        # sys.exit(1) 없음 → rc=0
```

이 패턴은 **MLOps 관점에서는 좋은 습관**이지만,  
exit code에만 의존하는 orchestrator에게는 "성공 신호 오염"을 일으킨다.

---

## 4. Solution Insight: Defense in Depth (다층 검증)

### 4.1 원칙

| 계층 | 신호 | 실패 시 처리 |
|------|------|------------|
| **L1** (Process) | `return_code == 0` | → 실패 경로 (분석·수정·재시도) |
| **L2** (Semantic) | `result.json.success == True` | → 실패 경로 (rc=0이어도) |
| **L3** (Content) | 숫자 metric 1개 이상 존재 | → 경고 (advisory, 실패 처리는 하지 않음) |

L2가 핵심이다. `result.json`은 스크립트가 자신의 성공 여부를 알고 명시적으로 기록한 것이므로 orchestrator가 반드시 신뢰해야 한다.

### 4.2 수정된 의사결정 트리

```
rc == 0?
  ├─ YES → result.json.success == True?
  │          ├─ YES → numeric metric 있음? → ✅ 성공
  │          │                없음 → ⚠️ 경고 + ✅ 성공 (L3 advisory)
  │          └─ NO  → 실패 경로 (rc=-3 synthetic) ← 핵심 수정
  └─ NO  → 실패 경로
```

### 4.3 구현

```python
if run_result["return_code"] == 0:
    metrics = run_result.get("result_json", {}) or {}
    
    if metrics.get("success", True):
        # L3: metric 존재 여부 advisory 체크
        has_numeric = any(isinstance(v, (int, float)) for k, v in metrics.items() if k != "success")
        if not has_numeric:
            emit("AGENT_MESSAGE", "[Phase 3] Warning: no numeric metrics in result.json", ...)
        
        return ExecutorResult(success=True, ...)  # 정상 성공 경로
    
    # L2: success=False → 실패 경로로 전환
    error_msg = metrics.get("error", "result.json.success=False")
    emit("AGENT_MESSAGE", f"[Phase 3] rc=0 but success=False: {error_msg}. Treating as failure.", ...)
    run_result = {**run_result, "return_code": -3, "stderr_tail": error_msg}

# 실패 경로 (rc != 0 또는 L2 실패)
stderr = run_result["stderr_tail"]
...
```

---

## 5. 추가 권고사항

### 단기 (즉시 적용)
- [x] L2 검증 적용 (`result.json.success=False` → 실패 처리)
- [x] L3 advisory warning 추가

### 중기 (다음 버전)
- **Verifier 에이전트 도입**: 실험 완료 후 result.json + stdout을 읽어 "실험이 실제로 의미 있는 결과를 냈는가"를 LLM이 판정하는 독립 에이전트
- **Metric schema validation**: `result.json`이 사전 정의된 스키마(primary metric, run_id, timestamp)를 충족하는지 검사

### 장기 (아키텍처 수준)
- **Process Supervision 도입**: 각 epoch마다 loss/accuracy를 스트리밍으로 수집하고, 이것이 실제로 감소하는지 실시간 모니터링
- **Outcome Reward Model 대신 Process Reward Model**: 단계별 중간 결과를 검증 기준으로 삼는 평가 구조 도입

---

## 6. 참고 문헌

1. Krakovna, V. et al. (2020). *Specification gaming: the flip side of AI ingenuity.* DeepMind Blog.
2. Lightman, H. et al. (2023). *Let's Verify Step by Step.* OpenAI. arXiv:2305.20050
3. Kinniment, M. et al. (2023). *Evaluating Language-Model Agents on Realistic Autonomous Tasks.* ARC Evals. arXiv:2312.11671
4. Liu, X. et al. (2023). *AgentBench: Evaluating LLMs as Agents.* arXiv:2308.03688
5. Krakovna, V. (2018). *Specification gaming examples in AI.* GitHub compendium.
