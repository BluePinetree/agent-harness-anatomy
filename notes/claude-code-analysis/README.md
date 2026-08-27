# Claude Code 아키텍처 분석 (2026-04)

> **무엇인가**: LLM 코딩 에이전트가 내부적으로 어떻게 구성되는지 직접 읽고 분석한 기록.
> MARS 설계에 영향을 준 학습 자료이며, **이 프로젝트의 방향 전환 배경 중 하나**입니다.

## 출처와 정리 경위

원래 참조했던 소스 트리는 자체 `LICENSE` 파일에
*"leaked proprietary source code belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"*
이라고 명시돼 있었습니다. 2026-08-20에 다음과 같이 정리했습니다.

| 조치 | 내용 |
|---|---|
| 소스 트리 삭제 | `claude-code-main/` (2,169 파일 / 39MB) 전체 제거. **커밋 이력에는 없었음(0건)** |
| 원문 코드 발췌 제거 | 문서 5개에서 TypeScript 블록 **46개** 제거 |
| 남긴 것 | 본인이 쓴 산문 분석과 직접 그린 구조도 (약 2,600줄) |

공식 문서: <https://docs.claude.com/en/docs/claude-code>

## 문서

| 파일 | 내용 | 줄수 |
|---|---|---|
| `ARCHITECTURE_KO.md` | 모듈 구조 4개 구역으로 나눠 분석 — 진입점·코어 엔진 / 툴 시스템 / 서비스·MCP·멀티에이전트 / UI·상태·설정 | 765 |
| `SCENARIO_KO.md` | 엔드투엔드 추적: 데이터 정리 후 파일 저장 (결과물 생성형 작업) | 465 |
| `SCENARIO_GITHUB_KO.md` | 외부 서비스 연동 흐름 — 연결 라이프사이클 · 도구명 정규화 · 원격 호출 | 495 |
| `SCENARIO_PROFILING_KO.md` | 실행 도구를 **진단 도구로** 쓰는 흐름 — 측정 → 파싱 → 원인 특정 → 보고 | 460 |
| `SCENARIO_WEBFETCH_KO.md` | 외부 문서 수집 → 변환 → 저장 흐름, 권한 확인 포함 | 419 |

## MARS 설계에 남긴 것

이 분석에서 가져온 판단들:

- **하네스 계층은 우리가 만들 자리가 아니다** — 도구 루프·권한·컨텍스트 관리는 이미 성숙함
- **비동기 제너레이터 기반 단일 루프**가 다중 에이전트 조율보다 단순하고 견고하다
- 그래서 MARS는 직선 파이프라인을 유지하고, 병렬은 방법론 탐색 한 곳에만 둔다

관련: `docs/new_project_design_ko.md`, `docs/landscape_watch_ko.md`
