# Research System — Project Guide

## 개요
AI 에이전트 프레임워크(CrewAI, AutoGen, LangGraph)를 비교 연구하는 프로젝트.
각 프레임워크로 동일한 연구 워크플로우를 구현해 성능/구조를 분석한다.

## 디렉토리 구조
```
research_system/
├── crewai_prototype/     # CrewAI 기반 구현
├── autogen_prototype/    # AutoGen 기반 구현
├── langgraph_prototype/  # LangGraph 기반 구현
├── research_system_ui/   # 공통 UI
```

## 공통 워크플로우
모든 프로토타입은 동일한 파이프라인을 구현:
`planner → designer → coder → executor → analyzer → writer`

## 코딩 규칙
- 에이전트 간 전달값은 JSON 구조체로 — 긴 텍스트 그대로 넘기지 않는다
- 실행 로그는 파일로 저장하고, 다음 에이전트엔 파일 경로만 전달
- LLM 호출 프롬프트는 간결하게 유지 (토큰 절약)
- 재현 가능성을 위해 가능하면 seed 고정

## 컨텍스트 관리
- 런타임 상태: `context/runtime_memory.md`
- 에이전트 간 핸드오프: `context/handoff_state.json`
- 컨텍스트가 오염됐다 싶으면 위 파일들 초기화 후 재시작

## 작업 시 주의사항
- 각 프레임워크 디렉토리 안에 개별 CLAUDE.md가 있으니 해당 디렉토리 작업 시 참고
- 프레임워크 간 비교 분석 문서는 루트의 `*_ko.md` 파일 참고
