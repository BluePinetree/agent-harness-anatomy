# 사용자 제공 데이터 인제스천 + tool 능동 활용 — 전문가 회의 설계 (2026-07-28)

주제: "다운 안 되는 데이터셋 → 사용자 수동 다운로드 가이드 → 폴더 배치 → 시스템이 폴더 분석/압축해제/경로 맞춘 실험코드 자동 작성 (tool 적극 활용)" 실현 가능성.
참석: 데이터 인제스천 · 에이전트 tool-use 아키텍트 · 파이프라인 HITL/UX.

## 결론: **실현 가능 (YES)**. 공수 M~L. 단, 전제 배선 1건이 모든 것의 열쇠.

## 세 전문가의 수렴점 (핵심)
**`data_path`/`data_description`이 API `ResearchRequest`엔 있으나 파이프라인에 전달되지 않는다.**
- `PreparedRun`·`scaffold_input`·planner/designer/codegen 프롬프트·실행 CLI 어디에도 안 감(오직 `input_normalizer`의 CIFAR-100 하드코딩 특례만 예외).
- 그런데 **수신부는 이미 준비됨**: 스캐폴드 CLI에 `--data_root`/`--data_path`/`--target_column`/`--timestamp_column` 존재, profile 자동선택기 존재, Phase3 escalation `GuidanceGate`(경로 회신→재개) 존재, 재사용 tool-loop(`rsp/tool_loop.py`) 존재.
- ⇒ **끊긴 배선(orchestrator→프롬프트, phase3→CLI)을 잇는 게 모든 커스텀 데이터 시나리오(S3/S6/S8/S10)와 수동배치 흐름의 공통 전제.**

## 통합 설계 (세 전문가 종합)
```
Phase 0  setup_workspace
  └▶ [신설] Phase 0b  DATA PROBE  ← 사용자의 tool 적극 활용 요구 지점
        대상: data_path(사용자 폴더) 또는 DATA_DIR
        - 읽기전용 조사 도구로 폴더 능동 조사: list_dir / read_head / probe_dataset / detect+extract_archive
        - 압축(zip/tar/gz) 자동 감지·해제(Zip-Slip 방어)
        - 구조 추론: ImageFolder(class 목록) / CSV(헤더·target·구분자) / timeseries(날짜컬럼)
        - 산출: dataset_manifest.json → handoff_dir 저장(경로만 다운스트림 전달)
        ├─ 데이터 없음 & 다운로드 실패(network) → DATA_NEEDED 게이트
        │     "직접 받아 <이 폴더>에 넣으세요 + 정확한 해제 구조" 안내 → 사용자 경로 회신 → 재-probe
  └▶ Preflight  ("데이터 감지: <manifest 요약>, target 컬럼 맞나요?" 확인 문항)
  └▶ Phase 1 planner/designer ← manifest 주입(실측 구조; 하드코딩 _DOMAIN_KNOWLEDGE 보완)
  └▶ Phase 2 coder ← _generate_content DATASET 섹션에 manifest 경로/스키마 주입
        (쓰기는 기존 LLM.call→Python write 유지 — 조사만 tool 활용)
  └▶ Phase 3 execute ← _experiment_cmd에 --data_path/--data_root/--target_column 채워 전달
        + 재현성: repro_env에 dataset_provenance{source:manual, path, size/sha, placed_at} 기록
```

## 핵심 아키텍처 판단 (tool-use 아키텍트)
- **"조사(read)"와 "쓰기(write)"를 분리하라.** Phase 2가 CrewAI 도구를 뺀 건 *쓰기 신뢰성*(LLM이 WriteTool 미호출→파일 미기록) 때문이지 읽기 때문이 아니다. **읽기(조사)는 도구 미호출해도 무해(재시도 가능)** → tool-loop이 딱 맞고, 과거 "도구 거부" 실패에서 구조적으로 자유롭다. 쓰기는 지금 방식 유지.
- **도구 2계열 분리 주의:** CrewAI `BaseTool`(crew_tools/)은 `rsp/tool_loop`에 못 꽂힘(스키마 메서드 부재). 조사 도구는 LangGraph식 경량 클래스(`to_anthropic_schema()`+`run()`)로 작성해 `run_tool_loop` 재사용.
- **샌드박스 확장:** 현 `WorkspaceListTool`은 workspace 안으로 제한 → 사용자 데이터 폴더(밖)를 읽으려면 **읽기전용·화이트리스트(data_root+workspace) `_resolve` 완화** 필요.

## 재사용 vs 신규
| 재사용 | 신규 |
|---|---|
| GuidanceGate/Registry, `/guidance` 엔드포인트(경로 회신→재개), preflight 선례, scaffold CLI 인자, profile 선택기, `rsp/tool_loop`, `apply_budget`, RunCommandTool 자산 | Phase 0b `data_probe`, 조사도구 4~5종(LangGraph식), `DatasetIngestor`/manifest, `DATA_NEEDED` 이벤트, `_is_data_download_failure()`(network 마커), **data_path 배선(orchestrator→프롬프트→CLI)**, GuidanceDrawer 분기 |

## 리스크 & 완화
- **Zip-Slip/심볼릭링크 탈출**: 해제 경로 resolve 후 prefix 검증, 링크 멤버 거부.
- **경로 신뢰**: 재개 전 폴더 존재·최소파일 검증, 실패 시 게이트 재개방("검증 실패" 표시).
- **대용량 폴더**: list_dir depth/max_entries 상한 + apply_budget 절단.
- **torchvision 캐시 레이아웃 민감성**: 안내에 정확한 해제 후 디렉토리 구조 명시(예: `cifar-10-batches-py/`).
- **헤드리스(anchor_run)**: 사람 없음 → DATA_NEEDED도 auto-skip = **정직한 실패**(가짜 성공 L4 원칙 일치). 실행 전 DATA_DIR 존재검사로 왕복 회피.
- **가짜 성공 회귀**: network 감지가 L4(degenerate)보다 **먼저** 걸리도록 rc 분기 순서.
- **프로바이더 결합(중)**: `rsp/tool_loop`은 Anthropic 전용. 파이프라인은 native-OpenAI(llm_factory). → 조사 엔진 모델을 **명시 상수로 고정**(벤치 공정성) 하거나 tool-loop을 llm_factory tool-use로 일반화(추가 M). **결정 필요.**

## 공수 (분해)
- **[전제] data_path 배선**(orchestrator→PreparedRun/scaffold_input→프롬프트→phase3 CLI): **M** — 모든 커스텀 데이터의 열쇠, 이것만으로 S3/S6/S8/S10 기본 동작.
- 조사도구 4~5종 + Phase 0b tool-loop(manifest 산출): **M**
- DATA_NEEDED 게이트 + network 감지 + GuidanceDrawer 분기: **M** (핵심 배선은 위 전제와 공유)
- 재현성 dataset_provenance: **S**
- 검증(S3/S6/S8/S10 회귀 + 오탐): **M**
합계 **M~L**.

## 권장 순서
1. ✅ **[P0 완료 2026-07-28] data_path 배선** — orchestrator→goal(planner/coder 전파)+phase3 `--data-path`/`--data-root`+DATA_DIR. 실제 스캐폴드 main.py 수용 확인.
2. ✅ **[P0 완료 2026-07-28] DATA_NEEDED 게이트 + network 감지** — `_is_data_download_failure`→DATA_NEEDED 이벤트/GuidanceGate→경로 회신→재개. 헤드리스=정직한 실패. (단위 검증; 라이브 E2E는 네트워크 회복 시.)
3. ✅ **[P1 완료 2026-07-28] Phase 0b tool-driven probe(결정론적 엔진)** — `orchestration/dataset_ingestor.py`: 도구 함수(list_tree/read_head/detect_archives/extract_archive/probe_dataset) + `ingest()` 드라이버. 폴더 조사→압축 자동해제(Zip-Slip 방어)→구조추론(ImageFolder/CSV/시계열/parquet/npz)→`dataset_manifest.json`. orchestrator Phase 0b에서 실행→manifest를 goal에 주입(planner/coder)+해제된 resolved_path를 phase3 data_path로 반영+handoff에 저장. 검증: 합성 폴더 5종(imagefolder/tabular/timeseries/archive/zip-slip) 전부 통과.
4. **[P1 잔여] (a) LLM tool-loop 계층** — 조사도구를 LLM 에이전트가 능동 호출(구조 불명/복합 폴더용). 네트워크 회복 후 착수(같은 도구 재사용, `rsp/tool_loop`). **(b) GuidanceDrawer DATA_NEEDED 응답 UI. (c) 재현성 provenance(manifest sha/size).**

## 구현 현황 (2026-07-28)
- ✅ P0-A data_path 배선 / P0-B DATA_NEEDED 게이트 / P1 Phase 0b 결정론적 조사엔진 — 구현·단위검증.
- ⏳ 잔여: LLM tool-loop(에이전트 능동 도구호출), DATA_NEEDED 웹 응답 UI, 라이브 E2E(네트워크 회복 시).

기록: `crewai_prototype/CLAUDE.md` "사용자 제공 데이터셋(data_path)…" 항목.
