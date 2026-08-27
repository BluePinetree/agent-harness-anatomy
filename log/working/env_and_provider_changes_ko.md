# 실행 환경 · LLM 프로바이더 변경 기록

작성일: 2026-07-27 · 대상: `crewai_prototype/`

## 1. litellm 우회 제거 → crewai 네이티브 프로바이더 (`core/llm_factory.py`)
- **변경**: `LLM(is_litellm=True, ...)` 강제(2곳)와 `num_retries` 주입을 제거. 이제 `LLM(**llm_kwargs)`로 **crewai 1.x 네이티브 OpenAI 프로바이더**를 그대로 사용.
- **배경**: 과거 `num_retries`/`parallel_tool_calls`를 llm_kwargs에 주입 → 네이티브 프로바이더가 이를 openai>=2.x SDK로 전달해 오류(TypeError/400) → 임시로 `is_litellm=True`로 litellm 경로 우회했었음. 두 파라미터를 애초에 주입하지 않으면 네이티브 경로가 정상 동작하므로 우회 제거.
- **검증**: MARS 환경 격리 호출에서 `LLM type=OpenAICompletion, is_litellm=False`로 "PONG" 확인. 파이프라인 Phase 1(planner/designer)·Phase 2(codegen)가 네이티브 프로바이더로 정상 실행.
- `requirements.txt`: `crewai[tools]==1.14.3`로 핀 수정(기존 `<1.0.0`는 실제와 불일치였음).

## 2. 능력 기반 실행 환경 보장 + 선택 메뉴 (`harness/anchor_run.py`)
특정 이름(MARS)에 하드코딩하지 않고, **파이프라인 실행에 필요한 패키지를 갖춘 "적합한" 가상환경**을 능력으로 판정한다.
- 필요 모듈: `crewai, torch, sklearn, pandas, dotenv` (전부 import 가능해야 적합). → 예: base는 torch가 깨져 부적합, MARS는 적합.
- 동작(파이프라인/crewai import "전에" 수행):
  1. 현재 환경 적합 + 비대화형 + `--env` 미지정 → 즉시 진행(탐색 생략).
  2. 그 외 → 설치된 conda 환경 탐색해 적합 목록 수집.
     - `--env <이름|경로>` 지정 → 해당 환경 사용
     - 적합 1개 → 그 환경 사용
     - **대화형 + 적합 2개↑ → 번호 선택 메뉴(1,2,3…)로 사용자가 선택**
     - 비대화형 + 2개↑ → 현재(적합 시) 또는 첫 적합 환경 자동 선택
  3. 적합 환경 0개 → **생성 방법 안내 후 종료**(conda create + torch cu118 + requirements + timm/seaborn).
  4. 선택 환경이 현재와 다르면 그 환경으로 **재실행(subprocess 대기)**.
- **검증**: MARS=탐색 없이 진행 / base=탐색→MARS 재실행 / `--env MARS`=명시 재실행 / 번호 메뉴(stdin 입력)=정상 선택 — 전부 확인.

## 3. os.execv Windows 분리(detach) 문제 수정
- Windows에서 `os.execv`는 프로세스를 진짜 대체하지 않고 **새 프로세스를 띄운 뒤 원본을 종료** → 자식이 분리되어 모니터링/로그가 끊기고, 분리된 자식이 Phase 3 Popen 교착으로 멈추는 사례 발생.
- **수정**: 재실행을 `subprocess.run([...], env=...)` + `sys.exit(rc)`로 변경 → 부모가 자식 종료까지 대기, 종료코드 전파, 로그/모니터링 정상화.

## 4. 최종 조정 — 재실행 대신 "직접 실행 안내"
- `os.execv`→`subprocess.run` 재실행으로 바꿨으나, **재실행 컨텍스트에서 Phase 3 실험 서브프로세스 스트리밍이 교착**(직접 실행은 정상)되는 문제 확인.
- 따라서 "적합한 다른 환경을 찾으면 **자동 재실행하지 않고, 그 환경에서 직접 실행할 명령을 안내하고 종료(exit 3)**"로 변경. (감지→안내→사용자 실행 = "유도" 취지 + 교착 회피)
- 현재 환경이 적합하면 그대로 실행. 적합 환경이 여럿이면 번호 선택 메뉴로 고른 뒤 그 환경 명령 안내.

## 검증 결과
- **native provider E2E 검증됨**: MARS 직접 실행 run에서 Phase 1(planner/designer)·Phase 2(coder)·Phase 4(writer) **모든 LLM 단계가 native provider로 정상 실행**되어 paper.md까지 생성(로그에 litellm/num_retries/parallel_tool 오류 전무). 실험 결과 자체는 stochastic codegen 변동성으로 실패할 수 있으나 provider와 무관.
- UI "선택지 → 선택 → 진행" 패턴 검토·구현: `docs/ui_option_selection_campaign_ko.md`.
