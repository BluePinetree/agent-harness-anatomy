# 시나리오: GitHub 이슈 연동 (MCPTool)

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

> **핵심 컴포넌트**: MCPTool — 외부 서비스와 Claude Code를 연결하는 브리지
> 이 시나리오에서만 등장하는 흐름: MCP 연결 라이프사이클 → 도구명 정규화 → JSON-RPC 호출

---

## 시나리오 설정

> **사용자 입력**: `"이번 주 GitHub 이슈 목록 가져와서 우선순위별로 정리해줘"`

Claude Code가 직접 GitHub API를 호출하는 것이 아니라,
**MCP 서버(GitHub Connector)** 를 통해 데이터를 받아오고 처리하는 흐름을 추적한다.

```
통과 순서:
MCP 연결 초기화 → MCPTool(list_issues) → MCPTool(get_issue) ×n →
Write(정리 문서 생성) → 최종 응답
```

---

## 사전 조건: MCP 서버 연결 (앱 시작 시)

**이 단계는 사용자가 입력하기 전에 이미 완료되어 있다.**

`.mcp.json` 또는 `settings.json`에 GitHub MCP 서버가 등록된 경우:

```json
// .mcp.json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

**MCP 연결 라이프사이클 (main.tsx 시작 시):**

```
1. getAllMcpConfigs()
   └─> .mcp.json에서 "github" 서버 설정 발견

2. useManageMCPConnections()
   └─> ensureConnectedClient("github")
       ├─ 트랜스포트 생성: StdioClientTransport
       │   → npx @modelcontextprotocol/server-github 프로세스 스폰
       ├─ client.initialize()
       │   → 서버 버전, 기능 협상
       └─ client.listTools()
           → 서버가 제공하는 도구 목록 수신

3. 도구명 정규화
   normalizeNameForMCP("github") → "github"
   buildMcpToolName("github", "list_issues")
   → "mcp__github__list_issues"

4. AppState.mcp 업데이트:
   {
     connections: [{ name: "github", type: "connected" }],
     tools: [
       { name: "mcp__github__list_issues",    ... },
       { name: "mcp__github__get_issue",       ... },
       { name: "mcp__github__create_issue",    ... },
       { name: "mcp__github__list_pull_requests", ... },
       // ... 총 12개 MCP 도구
     ]
   }
```

**이 시점의 AppState:**
> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 0: 앱 시작 (main.tsx)

```
git status
└─> "On branch main  /  nothing to commit"

도구 풀 조립 → getTools()
└─> [GrepTool, FileReadTool, FileEditTool, BashTool, ...]
    + MCPTool 자동 포함 (MCP 연결 완료된 도구들)
    총 12개 기본 + mcp__github__* 12개 = 24개
```

---

## STEP 1: 사용자 입력 접수 (QueryEngine)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 2: 첫 번째 API 호출 (턴 1)

**API에 보내는 도구 목록 (일부):**
```
tools: [
  { name: "Bash", ... },
  { name: "Read", ... },
  { name: "mcp__github__list_issues",
    description: "List issues from a GitHub repository",
    input_schema: {
      type: "object",
      properties: {
        owner:  { type: "string" },
        repo:   { type: "string" },
        state:  { type: "string", enum: ["open","closed","all"] },
        since:  { type: "string", description: "ISO 8601 date" },
        labels: { type: "string" }
      }
    }
  },
  { name: "mcp__github__get_issue", ... },
  // ...
]
```

**Claude 내부 thinking:**
```
"GitHub 이슈를 가져오려면 mcp__github__list_issues를 써야 한다.
 레포 정보(owner/repo)가 필요한데, 로컬 git remote에서 알 수 있다.
 먼저 git remote로 레포 정보를 확인하자."
```

**결정한 도구:**
```json
{
  "tool": "Bash",
  "id": "tu-001",
  "input": { "command": "git remote get-url origin" }
}
```

**[중간 결과 ①] 레포 정보:**
```
https://github.com/mycompany/my-project.git
returnCode: 0
```

---

## STEP 3: MCPTool — 이슈 목록 조회 (턴 2)

**Claude의 판단:**
```
"owner: mycompany, repo: my-project
 이번 주 = 2026-04-03 이후 (오늘 2026-04-09 기준 월요일)
 open 이슈만 가져오자."
```

**도구 호출:**
```json
{
  "tool": "mcp__github__list_issues",
  "id": "tu-002",
  "input": {
    "owner": "mycompany",
    "repo":  "my-project",
    "state": "open",
    "since": "2026-04-03T00:00:00Z"
  }
}
```

**MCPTool 내부 처리:**
```
1. 도구명에서 서버 추출
   "mcp__github__list_issues" → serverName: "github"

2. AppState.mcp.connections에서 서버 찾기
   → { name: "github", type: "connected", client: Client }

3. JSON-RPC 호출 (stdio 트랜스포트)
   → MCP 서버 프로세스에 전송:
   {
     "jsonrpc": "2.0",
     "method": "tools/call",
     "params": {
       "name": "list_issues",
       "arguments": {
         "owner": "mycompany",
         "repo": "my-project",
         "state": "open",
         "since": "2026-04-03T00:00:00Z"
       }
     }
   }

4. MCP 서버가 GitHub REST API 호출
   GET https://api.github.com/repos/mycompany/my-project/issues
   Authorization: Bearer ${GITHUB_TOKEN}

5. 결과 반환
```

**[중간 결과 ②] MCPTool 반환값:**
```json
[
  { "number": 142, "title": "로그인 페이지 500 에러",
    "labels": ["bug", "critical"], "created_at": "2026-04-07T09:12:00Z",
    "comments": 5, "assignee": "alice" },
  { "number": 141, "title": "다크모드 텍스트 가독성 개선",
    "labels": ["enhancement", "UI"], "created_at": "2026-04-05T14:30:00Z",
    "comments": 2, "assignee": null },
  { "number": 140, "title": "CSV 내보내기 인코딩 버그",
    "labels": ["bug"], "created_at": "2026-04-04T11:00:00Z",
    "comments": 8, "assignee": "bob" },
  { "number": 139, "title": "API 응답 캐싱 도입",
    "labels": ["performance", "enhancement"], "created_at": "2026-04-03T16:45:00Z",
    "comments": 3, "assignee": null },
  { "number": 138, "title": "테스트 커버리지 80% 달성",
    "labels": ["testing"], "created_at": "2026-04-03T10:20:00Z",
    "comments": 1, "assignee": "carol" }
]
```

---

## STEP 4: MCPTool — 상세 내용 조회 (턴 3)

**Claude의 판단:**
```
"critical 라벨 붙은 #142 이슈가 가장 중요할 것 같다.
 본문 내용도 확인해서 우선순위 판단에 활용하자."
```

**도구 호출 (2개 병렬):**
```json
[
  { "tool": "mcp__github__get_issue", "id": "tu-003",
    "input": { "owner": "mycompany", "repo": "my-project", "issue_number": 142 } },
  { "tool": "mcp__github__get_issue", "id": "tu-004",
    "input": { "owner": "mycompany", "repo": "my-project", "issue_number": 140 } }
]
```

**StreamingToolExecutor: 2개 MCP 호출 병렬 실행**

**[중간 결과 ③-A] 이슈 #142 상세:**
```json
{
  "number": 142,
  "title": "로그인 페이지 500 에러",
  "body": "## 재현 방법\n1. /login 접속\n2. 소셜 로그인 클릭\n3. 500 Internal Server Error\n\n## 영향 범위\n신규 가입 불가 상태. 프로덕션 영향 중.",
  "labels": ["bug", "critical"],
  "comments": 5,
  "assignee": { "login": "alice" }
}
```

**[중간 결과 ③-B] 이슈 #140 상세:**
```json
{
  "number": 140,
  "title": "CSV 내보내기 인코딩 버그",
  "body": "## 증상\nCSV 내보내기 시 한글이 깨짐\n\n## 환경\nWindows Excel에서만 발생, Mac은 정상",
  "labels": ["bug"],
  "comments": 8
}
```

---

## STEP 5: Write — 정리 문서 생성 (턴 4)

**Claude의 우선순위 분류 로직:**
```
P1 (즉시): critical 라벨 + 프로덕션 영향 → #142
P2 (이번 주): bug 라벨 + 댓글 많음 → #140
P3 (다음 주): enhancement, 담당자 없음 → #141, #139
P4 (여유 시): testing → #138
```

**도구 호출:**
```json
{
  "tool": "Write",
  "id": "tu-005",
  "input": {
    "file_path": "docs/issues-2026-04-w2.md",
    "content": "..."
  }
}
```

**Write 도구가 생성하는 파일:**
```markdown
# GitHub 이슈 현황 — 2026년 4월 2주차

> 조회 기간: 2026-04-03 ~ 2026-04-09  |  열린 이슈: 5건

---

## 🔴 P1 — 즉시 처리 (1건)

| # | 제목 | 담당자 | 댓글 |
|---|------|--------|------|
| [#142](https://github.com/mycompany/my-project/issues/142) | 로그인 페이지 500 에러 | alice | 5 |

**상황**: 소셜 로그인 클릭 시 500 에러 발생. 신규 가입 불가 상태.
프로덕션 영향 중 — 최우선 처리 필요.

---

## 🟠 P2 — 이번 주 처리 (1건)

| # | 제목 | 담당자 | 댓글 |
|---|------|--------|------|
| [#140](https://github.com/mycompany/my-project/issues/140) | CSV 내보내기 인코딩 버그 | bob | 8 |

**상황**: Windows Excel에서 한글 깨짐. Mac은 정상.
댓글이 많아 이미 여러 팀원이 인지한 상태.

---

## 🟡 P3 — 다음 주 처리 (2건)

| # | 제목 | 담당자 | 댓글 |
|---|------|--------|------|
| [#141](https://github.com/mycompany/my-project/issues/141) | 다크모드 텍스트 가독성 개선 | 미배정 | 2 |
| [#139](https://github.com/mycompany/my-project/issues/139) | API 응답 캐싱 도입 | 미배정 | 3 |

---

## 🟢 P4 — 여유 시 처리 (1건)

| # | 제목 | 담당자 | 댓글 |
|---|------|--------|------|
| [#138](https://github.com/mycompany/my-project/issues/138) | 테스트 커버리지 80% 달성 | carol | 1 |

---

*생성: 2026-04-09 / Claude Code 자동 정리*
```

**권한 확인:**
```
┌────────────────────────────────────────────────┐
│  Create docs/issues-2026-04-w2.md              │
│                                                │
│  # GitHub 이슈 현황 — 2026년 4월 2주차          │
│  > 조회 기간: 2026-04-03 ~ ...                  │
│  ...                                           │
│                                                │
│  [y] 허용   [n] 거부                            │
└────────────────────────────────────────────────┘
사용자: y
```

**[중간 결과 ④] 파일 생성 완료:**
```
{ filePath: "docs/issues-2026-04-w2.md", bytesWritten: 1243 }
```

---

## STEP 6: 최종 응답 (턴 5)

```
완료했습니다. docs/issues-2026-04-w2.md 에 정리했습니다.

이번 주 열린 이슈 5건을 우선순위별로 분류했습니다:

🔴 P1 (즉시): #142 로그인 500 에러
  → 프로덕션 영향 중. alice 담당. 최우선 처리 필요.

🟠 P2 (이번 주): #140 CSV 한글 인코딩
  → 댓글 8개로 이미 여러 팀원이 인지. bob 담당.

🟡 P3 (다음 주): #141 다크모드, #139 API 캐싱
  → 담당자 미배정. 스프린트 계획 시 배정 권장.

🟢 P4 (여유 시): #138 테스트 커버리지
  → carol 담당. 장기 과제.
```

---

## 전체 타임라인

```
T+0.0s   "GitHub 이슈 정리해줘" 입력
T+0.1s   QueryEngine 진입
T+0.2s   [턴1] BashTool → git remote get-url (자동)
T+0.3s   owner/repo 파악: mycompany/my-project
T+0.5s   [턴2] MCPTool → mcp__github__list_issues
           JSON-RPC → GitHub API → 이슈 5건 반환 (820ms)
T+1.4s   [턴3] MCPTool × 2 병렬
           mcp__github__get_issue #142, #140 (동시, 610ms)
T+2.1s   [턴4] Write → docs/issues-2026-04-w2.md 생성
           권한 확인 → 사용자 y
T+5.5s   파일 생성 완료
T+5.6s   [턴5] 최종 응답 스트리밍
T+5.9s   출력 완료

────────────────────────────────────────────────
총 소요:    ~5.9초 (사용자 확인 ~3.5초 포함)
API 호출:   5턴
도구 실행:  5회 (Bash×1, MCP×3, Write×1)
MCP 호출:   3회 (list×1, get×2 병렬)
사용자 확인: 1회 (Write)
총 비용:    ~$0.0007
────────────────────────────────────────────────
```

---

## 컴포넌트별 역할

```
┌──────────────────────────────────────────────────────────────┐
│  사용자: "GitHub 이슈 가져와서 우선순위별로 정리해줘"          │
└──────────────────────┬───────────────────────────────────────┘
                       │
  [앱 시작 시 이미 완료]│
  .mcp.json 읽기       │
  → npx MCP서버 스폰   │
  → listTools()        │
  → AppState.mcp 등록  │
                       │
               ┌───────▼───────┐
               │  main.tsx     │  도구 24개 (기본 12 + MCP 12)
               └───────┬───────┘
                       │
               ┌───────▼───────┐
               │  QueryEngine  │  세션 초기화
               └───────┬───────┘
                       │
  ┌────────────────────▼──────────────────────────────────┐
  │                   queryLoop                            │
  │                                                        │
  │  턴1  BashTool            → owner/repo 확인 ①         │
  │  턴2  MCPTool             → list_issues × 5건 ②      │
  │       (JSON-RPC → GitHub REST API)                    │
  │  턴3  MCPTool × 2 병렬    → get_issue #142, #140 ③   │
  │  턴4  Write               → [y] 정리 문서 생성 ④      │
  │  턴5  최종 응답            → 우선순위 요약 출력        │
  └───────────────────────────────────────────────────────┘
                       │
               docs/issues-2026-04-w2.md
```

---

## MCPTool이 다른 도구와 다른 점

```
일반 도구 (BashTool, FileReadTool 등):
  Claude Code 프로세스 내에서 직접 실행
  → 로컬 파일시스템, 셸에 직접 접근

MCPTool:
  외부 MCP 서버 프로세스에 JSON-RPC로 위임
  → MCP 서버가 실제 작업 수행 (GitHub API 호출 등)
  → Claude Code는 결과만 받음

도구명 패턴:
  mcp__{서버명}__{기능명}
  "mcp__github__list_issues"
       ↑         ↑
    정규화된    MCP 서버가
    서버 이름   선언한 도구명
```

## 컴포넌트가 주고받은 데이터 한눈에

| 턴 | 도구 | 입력 | 출력 | 비고 |
|----|------|------|------|------|
| 1 | BashTool | `git remote get-url origin` | `github.com/mycompany/my-project` | 자동 허용 |
| 2 | MCPTool | `list_issues(owner, repo, since)` | 이슈 5건 JSON | JSON-RPC → GitHub API |
| 3 | MCPTool ×2 | `get_issue(142)`, `get_issue(140)` | 이슈 본문 2건 | **병렬 실행** |
| 4 | Write | 정리된 마크다운 전체 | `bytesWritten: 1243` | **사용자 y** |
| 5 | (없음) | — | 최종 요약 텍스트 | 스트리밍 |
