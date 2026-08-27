# 엔드투엔드 시나리오: 데이터 정리 후 엑셀 저장

> **출처 안내 (2026-08-20 이관)**
>
> 이 문서는 2026-04에 작성한 **본인의 아키텍처 분석 기록**입니다.
> 원래 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했고, **이 문서에 있던 원문 코드 발췌(총 2개 블록)도 함께 제거**했습니다.
>
> 남은 것은 전부 본인이 쓴 산문 분석과 직접 그린 구조도입니다.
> 공식 문서: https://docs.claude.com/en/docs/claude-code
>
> Status: finished · Written: 2026-04-08 · Sanitized: 2026-08-20

---

> **Claude Code 모듈 아키텍처 분석** 부속 문서
> 코드 수정이 아닌 **결과물 생성** 작업에서 각 컴포넌트가 주고받는 데이터 추적

---

## 시나리오 설정

> **사용자 입력**: `"sales 폴더에 이번 달 CSV 파일들 있는데, 상품별로 매출 합산해서 엑셀로 저장해줘"`

코드 버그 수정도, 기능 추가도 아닌 **데이터 처리 후 파일 출력** 작업.
Claude Code가 데이터 구조를 스스로 파악하고, 처리 스크립트를 직접 작성해 실행하는 흐름을 추적한다.

```
통과 순서:
GlobTool → FileReadTool(샘플 확인) → Write(스크립트 생성) →
BashTool(실행) → BashTool(결과 확인) → 최종 응답
```

---

## STEP 0: 앱 시작 (main.tsx)

```
git status
└─> "On branch main  /  nothing to commit"

CLAUDE.md
└─> (없음 — 특별한 프로젝트 가이드 없음)

도구 풀 조립
└─> [GlobTool, FileReadTool, FileEditTool, Write,
      BashTool, GrepTool, AgentTool, ...]  총 12개
```

**AppState 초기값:**
> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 1: 사용자 입력 접수 (QueryEngine)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 2: 첫 번째 API 호출 → 파일 탐색 (턴 1)

**Claude 내부 thinking:**
```
"먼저 sales 폴더에 파일이 몇 개, 어떤 이름으로 있는지 확인해야 한다.
 그 다음 CSV 구조를 파악하고 나서 처리 방법을 결정하자."
```

**결정한 도구:**
```json
{
  "tool": "Glob",
  "id": "tu-001",
  "input": {
    "pattern": "sales/*.csv",
    "path": "."
  }
}
```

**권한 확인:**
```
GlobTool.isReadOnly() = true → 자동 허용
```

**[중간 결과 ①] 파일 목록:**
```
{
  durationMs: 28,
  numFiles: 5,
  filenames: [
    "sales/2026-04-01.csv",
    "sales/2026-04-02.csv",
    "sales/2026-04-03.csv",
    "sales/2026-04-04.csv",
    "sales/2026-04-05.csv"
  ]
}
```

**Claude의 판단:**
```
"5개 파일이 있다. 구조가 어떻게 생겼는지 하나만 열어보자.
 컬럼 이름을 알아야 스크립트를 짤 수 있다."
```

---

## STEP 3: FileReadTool — 샘플 파일 구조 확인 (턴 2)

```json
{
  "tool": "Read",
  "id": "tu-002",
  "input": { "file_path": "sales/2026-04-01.csv" }
}
```

**[중간 결과 ②] CSV 첫 번째 파일 내용:**
```
날짜,상품명,카테고리,수량,단가,매출
2026-04-01,블루투스 이어폰,전자기기,3,29000,87000
2026-04-01,텀블러,주방용품,5,18000,90000
2026-04-01,블루투스 이어폰,전자기기,2,29000,58000
2026-04-01,노트북 파우치,액세서리,1,35000,35000
2026-04-01,텀블러,주방용품,8,18000,144000
2026-04-01,무선 마우스,전자기기,4,22000,88000
```

**Claude가 구조 파악 완료:**
```
"컬럼: 날짜, 상품명, 카테고리, 수량, 단가, 매출
 상품명이 여러 행에 걸쳐 중복됨.
 원하는 결과: 상품명 기준 매출 합산.

 처리 방법 선택:
 - Python pandas + openpyxl이면 5줄로 가능
 - pandas, openpyxl 있는지 확인 후 스크립트 작성"
```

---

## STEP 4: BashTool — 라이브러리 확인 (턴 3)

```json
{
  "tool": "Bash",
  "id": "tu-003",
  "input": { "command": "python3 -c \"import pandas, openpyxl; print('ok')\" 2>&1" }
}
```

**BashTool 권한 검사:**
```
1단계: AST 분석 → python3 -c 는 안전
2단계: 규칙 매칭 → 없음
3단계: 사용자 확인
  → "python3 -c \"import pandas...\" 실행하시겠습니까? [y/n]"
  사용자: y
```

**[중간 결과 ③] 라이브러리 확인 결과:**
```
ok
returnCode: 0
```

**Claude:**
```
"pandas, openpyxl 모두 있다. 스크립트 작성하자."
```

---

## STEP 5: Write — 처리 스크립트 생성 (턴 4)

코드를 수정하는 게 아니라 **새 파일을 처음부터 만든다.**
이 단계에서 FileEditTool 대신 **Write 도구**가 쓰인다.

```json
{
  "tool": "Write",
  "id": "tu-004",
  "input": {
    "file_path": "sales/summarize.py",
    "content": "..."
  }
}
```

**Write 도구가 생성하는 파일 내용:**
```python
# sales/summarize.py
import pandas as pd
import glob
import os
from datetime import datetime

# 1. sales/*.csv 전부 읽어서 합치기
csv_files = glob.glob("sales/*.csv")
csv_files = [f for f in csv_files if "summarize" not in f]  # 스크립트 자신 제외

df = pd.concat(
    [pd.read_csv(f) for f in csv_files],
    ignore_index=True
)

# 2. 상품별 집계
summary = (
    df.groupby(["상품명", "카테고리"])
    .agg(
        총수량=("수량", "sum"),
        총매출=("매출", "sum"),
        평균단가=("단가", "mean"),
        거래건수=("매출", "count"),
    )
    .reset_index()
    .sort_values("총매출", ascending=False)
)

# 3. 엑셀 저장 (서식 포함)
output_path = f"sales/매출요약_{datetime.today().strftime('%Y%m')}.xlsx"

with pd.ExcelWriter(output_path, engine="openpyxl") as writer:
    summary.to_excel(writer, index=False, sheet_name="상품별 매출")

    # 열 너비 자동 조정
    ws = writer.sheets["상품별 매출"]
    for col in ws.columns:
        max_len = max(len(str(cell.value or "")) for cell in col) + 2
        ws.column_dimensions[col[0].column_letter].width = max_len

print(f"저장 완료: {output_path}")
print(f"총 {len(summary)}개 상품, {len(df)}건 처리")
```

**권한 확인 — Write 도구는 새 파일 생성:**
```
┌───────────────────────────────────────────────────┐
│  Create sales/summarize.py                        │
│                                                   │
│  # sales/summarize.py                             │
│  import pandas as pd                              │
│  import glob                                      │
│  ...                                              │
│                                                   │
│  [y] 허용   [n] 거부                               │
└───────────────────────────────────────────────────┘
사용자: y
```

**[중간 결과 ④] 파일 생성 완료:**
```
{
  filePath: "sales/summarize.py",
  bytesWritten: 812
}
```

---

## STEP 6: BashTool — 스크립트 실행 (턴 5)

```json
{
  "tool": "Bash",
  "id": "tu-005",
  "input": { "command": "cd sales && python3 summarize.py 2>&1" }
}
```

**권한 확인:**
```
1단계: AST 분석 → cd + python3 실행, 위험 패턴 없음
2단계: 규칙 매칭 → 없음
3단계: 사용자 확인
  → "cd sales && python3 summarize.py 실행하시겠습니까? [y/n]"
  사용자: y
```

**내부에서 일어나는 일:**
```
pandas가 sales/*.csv 5개 파일을 읽음 (총 1,243행)
상품명 기준 groupby → 집계
openpyxl로 Excel 파일 생성 + 열 너비 자동 조정
```

**[중간 결과 ⑤] 실행 결과:**
```
저장 완료: sales/매출요약_202604.xlsx
총 12개 상품, 1243건 처리
returnCode: 0
durationMs: 1820
```

---

## STEP 7: BashTool — 결과 파일 확인 (턴 6)

Claude가 스스로 한 번 더 확인하기로 결정.

```json
{
  "tool": "Bash",
  "id": "tu-006",
  "input": { "command": "python3 -c \"import openpyxl; wb=openpyxl.load_workbook('sales/매출요약_202604.xlsx'); ws=wb.active; print(f'행 수: {ws.max_row-1}개'); [print(f'  {row[0].value}: {row[4].value:,}원') for row in list(ws.iter_rows())[1:6]]\"" }
}
```

**[중간 결과 ⑥] 상위 5개 상품 미리보기:**
```
행 수: 12개
  블루투스 이어폰: 1,856,000원
  텀블러: 1,404,000원
  무선 마우스: 1,232,000원
  노트북 파우치: 980,000원
  보조배터리: 847,000원
returnCode: 0
```

---

## STEP 8: 최종 응답 (턴 7)

**Claude 스트리밍 출력:**
```
완료했습니다.

sales/매출요약_202604.xlsx 파일을 생성했습니다.

처리 결과:
  - CSV 파일 5개 (2026-04-01 ~ 04-05) 합산
  - 총 1,243건 거래 → 12개 상품으로 집계

상위 5개 상품 (매출순):
  1위  블루투스 이어폰   1,856,000원
  2위  텀블러           1,404,000원
  3위  무선 마우스       1,232,000원
  4위  노트북 파우치       980,000원
  5위  보조배터리          847,000원

엑셀 파일에는 상품명, 카테고리, 총수량, 총매출, 평균단가, 거래건수
컬럼이 매출 내림차순으로 정렬되어 있습니다.
열 너비도 내용에 맞게 자동 조정했습니다.
```

---

## 전체 타임라인

```
T+0.0s   "CSV 합쳐서 엑셀로 저장해줘" 입력
T+0.1s   QueryEngine 진입
T+0.2s   [턴1] GlobTool → sales/*.csv 5개 발견          (28ms)
T+0.4s   [턴2] FileReadTool → CSV 구조 확인 (컬럼 파악) (11ms)
T+0.6s   [턴3] BashTool → pandas/openpyxl 확인          (340ms)
           권한 확인 → 사용자 y
T+4.0s   [턴4] Write → summarize.py 생성
           권한 확인 → 사용자 y
T+7.5s   파일 생성 완료
T+7.6s   [턴5] BashTool → python3 summarize.py 실행
           권한 확인 → 사용자 y
T+9.4s   실행 완료 (1,243건 처리, 1.8s)
T+9.5s   [턴6] BashTool → 결과 검증 (자동)
T+9.7s   상위 5개 미리보기 확인
T+9.8s   [턴7] 최종 응답 스트리밍
T+10.2s  출력 완료

────────────────────────────────────────────────────
총 소요:    ~10.2초 (사용자 확인 대기 ~7초 포함)
API 호출:   7턴
도구 실행:  6회 (Glob×1, Read×1, Bash×3, Write×1)
사용자 확인: 3회 (라이브러리 확인, 파일 생성, 스크립트 실행)
생성 파일:  2개 (summarize.py, 매출요약_202604.xlsx)
총 비용:    ~$0.0009
────────────────────────────────────────────────────
```

---

## 컴포넌트별 역할 요약

```
┌────────────────────────────────────────────────────────────────┐
│  사용자: "CSV 합쳐서 상품별 매출 엑셀로 저장해줘"               │
└──────────────────────────┬─────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  main.tsx   │  도구 조립 (Write 포함)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ QueryEngine │  세션 초기화
                    └──────┬──────┘
                           │
      ┌────────────────────▼─────────────────────────────────┐
      │                   queryLoop                           │
      │                                                       │
      │  턴1  GlobTool      → CSV 5개 파일 경로 ①            │
      │  턴2  FileReadTool  → 컬럼 구조 확인 ②               │
      │  턴3  BashTool      → [y] 라이브러리 존재 확인 ③     │
      │  턴4  Write         → [y] summarize.py 생성 ④        │
      │  턴5  BashTool      → [y] 스크립트 실행 ⑤            │
      │  턴6  BashTool      → 결과 검증 ⑥ (자동)             │
      │  턴7  최종 응답     → 요약 + 상위 5개 출력            │
      └───────────────────────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
   sales/summarize.py              sales/매출요약_202604.xlsx
   (중간 산출물)                    (최종 결과물)
```

---

## 기존 시나리오들과의 결정적 차이

| 항목 | 버그 수정 (장바구니) | 기능 추가 (마감일) | **데이터 처리 (엑셀)** |
|------|-------------------|-----------------|----------------------|
| 첫 행동 | Grep | Glob | **Glob** |
| 핵심 도구 | FileEditTool | FileEditTool | **Write + BashTool** |
| 목적 | 기존 코드 고치기 | 기존 코드에 추가 | **새 파일 만들기** |
| FileEditTool 사용 | 1회 | 5회 | **0회** |
| Write 사용 | 없음 | 없음 | **1회** |
| BashTool 목적 | 검증(tsc) | 검증(tsc) | **실제 작업 수행** |
| 결과물 | 코드 변경 | 코드 변경 | **데이터 파일(.xlsx)** |
| 사용자 확인 횟수 | 2회 | 1회('a') | **3회** (각 Bash마다) |

### 권한 패턴 차이

```
버그/기능 시나리오:
  첫 Edit에서 'a' → 이후 전부 자동
  (같은 종류의 작업이 반복되므로)

데이터 처리 시나리오:
  Bash 실행마다 매번 확인
  (실행 내용이 매번 다르고, 외부 효과가 있으므로)
```

### 핵심: Write vs FileEditTool

```
FileEditTool  → 이미 있는 파일의 특정 부분을 바꾼다
              → old_string / new_string 기반 정확한 매칭

Write         → 파일을 처음부터 새로 만든다
              → 전체 내용을 한 번에 기술
              → 존재하는 파일에 쓰면 덮어씀
```

---

## 컴포넌트가 주고받은 데이터 한눈에

| 턴 | 도구 | 입력 | 출력 | 비고 |
|----|------|------|------|------|
| 1 | GlobTool | `sales/*.csv` | 파일 경로 5개 | 자동 허용 |
| 2 | FileReadTool | `sales/2026-04-01.csv` | CSV 6개 컬럼, 6행 샘플 | 자동 허용 |
| 3 | BashTool | `python3 -c "import pandas..."` | `ok` / returnCode 0 | **사용자 y** |
| 4 | Write | `sales/summarize.py` + 30줄 코드 | `bytesWritten: 812` | **사용자 y** |
| 5 | BashTool | `python3 summarize.py` | `1243건 처리 완료` | **사용자 y** |
| 6 | BashTool | openpyxl로 결과 검증 | 상위 5개 상품·매출 | 자동 허용 |
| 7 | (없음) | — | 최종 요약 텍스트 | 스트리밍 출력 |
