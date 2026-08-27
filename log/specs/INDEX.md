# Research System V4 — 설계 문서 인덱스

> 두 설계자(30년 경력 시스템 엔지니어 + 30년 경력 알고리즘 전문가)의 설계 회의 결과물
> 작성일: 2026-05-19

---

## 핵심 설계 원칙

1. **절대 포기하지 않는다**: 서킷 브레이커 없음. 자동 수리 한도 소진 시 사용자에게 에스컬레이션.
2. **사용자 승인 게이트**: 플래닝 결과를 보여주고 명시적 승인 후에만 코딩 시작.
3. **단계별 코딩**: 파일을 한꺼번에 생성하지 않고 의존성 순서대로 Stage 1→2→3.
4. **섹션별 논문 작성**: Introduction→Related Works→Proposed Method→Experiments→Conclusion→References→Abstract 순서로 자체 검증하며 작성.
5. **사용자 상호작용**: 에러가 반복될 때 자동으로 멈추는 게 아니라 사용자에게 힌트 요청.

---

## 문서 목록

| 파일 | 담당 | 내용 |
|------|------|------|
| [DESIGN_MEETING.md](DESIGN_MEETING.md) | 통합 | 설계 회의 핵심 결론 및 구현 로드맵 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 시스템 엔지니어 | 전체 아키텍처, 모듈 구조, 기술 선택 근거 |
| [API_SPEC.md](API_SPEC.md) | 시스템 엔지니어 | REST API, Pydantic 모델, SSE 이벤트 스키마 |
| [MODULE_SPEC.md](MODULE_SPEC.md) | 시스템 엔지니어 | 모든 Python 모듈 클래스/함수 시그니처 |
| [PIPELINE_SPEC.md](PIPELINE_SPEC.md) | 알고리즘 전문가 | 파이프라인 상태 머신, Phase별 로직 |
| [ERROR_RECOVERY_SPEC.md](ERROR_RECOVERY_SPEC.md) | 알고리즘 전문가 | 에러 분류 체계, 복구 전략, 에스컬레이션 |
| [AGENT_SPEC.md](AGENT_SPEC.md) | 알고리즘 전문가 | 에이전트별 역할, 프롬프트 템플릿, 입출력 스키마 |
| [CONSTANTS.md](CONSTANTS.md) | 알고리즘 전문가 | 모든 설정값, 기본값, 튜닝 가이드 |

---

## 구현 파일 위치

신규 시스템은 `crewai_prototype/` 내부에 구현:

```
crewai_prototype/
├── orchestration/
│   ├── pipeline_orchestrator.py   # PipelineOrchestrator (핵심)
│   ├── approval_registry.py       # ApprovalGate, GuidanceRegistry
│   └── cancellation.py            # CancellationToken
├── phases/
│   ├── phase0_workspace.py        # WorkspaceSetupService
│   ├── phase1_planning.py         # PlannerDesignerService
│   ├── phase2_coding.py           # StagedCoderService (no circuit breaker)
│   ├── phase3_execution.py        # ExecutionService
│   └── phase4_writing.py          # WriterService
├── platform/
│   └── constants.py               # 모든 설정값
└── crew_tools/
    ├── syntax_check_tool.py       # SyntaxCheckTool
    └── import_check_tool.py       # ImportCheckTool
```
