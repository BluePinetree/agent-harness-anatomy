# 시나리오: 웹 조사 후 마크다운 저장 (WebFetchTool)

> **출처 안내 (2026-08-20 이관)**
>
> 이 문서는 2026-04에 작성한 **본인의 아키텍처 분석 기록**입니다.
> 원래 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했고, **이 문서에 있던 원문 코드 발췌(총 3개 블록)도 함께 제거**했습니다.
>
> 남은 것은 전부 본인이 쓴 산문 분석과 직접 그린 구조도입니다.
> 공식 문서: https://docs.claude.com/en/docs/claude-code
>
> Status: finished · Written: 2026-04-08 · Sanitized: 2026-08-20

---

> **핵심 컴포넌트**: WebFetchTool — 외부 URL을 읽어 마크다운으로 변환
> 이 시나리오에서만 등장하는 흐름: URL 권한 확인 → HTML→Markdown 변환 → 내용 기반 문서 생성

---

## 시나리오 설정

> **사용자 입력**: `"React Query랑 SWR 차이점 조사해서 팀 위키에 마크다운으로 저장해줘"`

코드도 건드리지 않고 외부 문서도 없는 상태에서,
**Claude Code가 직접 웹을 읽고, 내용을 정리해 파일로 저장**하는 흐름을 추적한다.

```
통과 순서:
WebFetchTool(React Query 공식) → WebFetchTool(SWR 공식) →
WebFetchTool(비교 글) → Write(정리 문서 생성) → 최종 응답
```

---

## STEP 0: 앱 시작 (main.tsx)

```
git status
└─> "On branch main  /  nothing to commit"

도구 풀 조립
└─> [GrepTool, FileReadTool, BashTool, WebFetchTool, Write, ...]  총 12개

AppState:
{
  settings: { model: "claude-sonnet-4-6" },
  toolPermissionContext: { mode: "default" },
  mcp: { connections: [] }
}
```

---

## STEP 1: 사용자 입력 접수 (QueryEngine)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 2: 첫 번째 API 호출 (턴 1)

**Claude 내부 thinking:**
```
"React Query와 SWR을 비교하려면 각각의 공식 문서에서
 핵심 철학, 주요 기능, API 차이를 읽어봐야 한다.
 두 공식 문서를 먼저 읽자."
```

**결정한 도구 (2개 동시):**
```json
[
  {
    "tool": "WebFetch",
    "id": "tu-001",
    "input": {
      "url": "https://tanstack.com/query/latest/docs/framework/react/overview",
      "prompt": "React Query의 핵심 특징, 설계 철학, 주요 기능을 요약해줘"
    }
  },
  {
    "tool": "WebFetch",
    "id": "tu-002",
    "input": {
      "url": "https://swr.vercel.app/docs/getting-started",
      "prompt": "SWR의 핵심 특징, 설계 철학, 주요 기능을 요약해줘"
    }
  }
]
```

**StreamingToolExecutor: 2개 WebFetch 병렬 실행**

---

## STEP 3: WebFetchTool 내부 처리 (×2 병렬)

**WebFetchTool 권한 확인:**

```
WebFetchTool.checkPermissions({ url: "https://tanstack.com/..." })

// 사전 승인 호스트 목록 확인
const preapprovedHosts = ["api.github.com", "github.com", "www.wikipedia.org", ...]

// tanstack.com → 목록에 없음 → 사용자에게 묻기
┌─────────────────────────────────────────────────────┐
│  WebFetch https://tanstack.com/query/latest/...     │
│  WebFetch https://swr.vercel.app/docs/...           │
│                                                     │
│  [y] 허용   [n] 거부   [a] 이 도메인 항상 허용       │
└─────────────────────────────────────────────────────┘
사용자: a  (이후 같은 도메인 자동 허용)
```

**WebFetchTool 내부 실행 순서:**
```
1. fetch(url)  →  HTML 수신
2. HTML → Markdown 변환 (getURLMarkdownContent)
   → 불필요한 nav, footer, script 제거
   → 본문 텍스트 + 코드 블록 보존
   → 최대 50,000자로 잘라냄 (MAX_MARKDOWN_LENGTH)
3. AI 프롬프트 적용 (applyPromptToMarkdown)
   → "핵심 특징, 설계 철학, 주요 기능을 요약해줘" 프롬프트로
      마크다운에서 핵심만 추출
4. 결과 반환
```

**[중간 결과 ①] React Query 공식 문서 요약:**
```markdown
## React Query 핵심

**설계 철학**: "서버 상태를 위한 데이터 동기화 라이브러리"
- 서버 상태와 클라이언트 상태를 명확히 분리
- 비동기 작업(fetch, mutation)을 선언적으로 관리

**주요 기능**:
- useQuery: 데이터 페칭 + 자동 캐싱
- useMutation: 서버 데이터 변경
- QueryClient: 전역 캐시 관리
- 자동 백그라운드 리패칭, Window focus 재검증
- DevTools 내장

**특징**: 캐시 무효화(invalidateQueries), 낙관적 업데이트,
무한 스크롤(useInfiniteQuery) 등 고급 기능 풍부
```

**[중간 결과 ②] SWR 공식 문서 요약:**
```markdown
## SWR 핵심

**설계 철학**: "Stale-While-Revalidate" — 캐시를 먼저 보여주고 백그라운드에서 갱신
- Vercel 팀이 Next.js 연동을 고려해 설계
- 단순함과 번들 크기 최소화 우선

**주요 기능**:
- useSWR: 단일 훅으로 페칭 + 캐싱
- useSWRMutation: 서버 변경
- 전역 설정: SWRConfig

**특징**: 번들 크기 4KB (React Query 13KB 대비 가벼움),
Next.js App Router와 자연스러운 통합
```

---

## STEP 4: WebFetchTool — 실무 비교 글 추가 조사 (턴 2)

**Claude의 판단:**
```
"공식 문서만으로는 실제 사용 시 차이가 불명확하다.
 번들 크기, DevTools 지원 같은 실무적 차이를 다룬 글도 하나 읽자."
```

```json
{
  "tool": "WebFetch",
  "id": "tu-003",
  "input": {
    "url": "https://npmtrends.com/react-query-vs-swr",
    "prompt": "다운로드 수, 번들 크기, 최근 트렌드 수치를 알려줘"
  }
}
```

**권한 확인:**
```
npmtrends.com → 사전 승인 목록 없음
→ "WebFetch https://npmtrends.com/... [y/n]"
사용자: y
```

**[중간 결과 ③] npm 트렌드 데이터:**
```
React Query (@tanstack/react-query):
  주간 다운로드: 8,200,000
  번들 크기: 13.1 KB (gzip)
  GitHub Stars: 41,000+

SWR:
  주간 다운로드: 3,100,000
  번들 크기: 4.2 KB (gzip)
  GitHub Stars: 30,000+

트렌드: React Query가 2023년 이후 격차 더 벌어지는 중
```

---

## STEP 5: Write — 비교 문서 생성 (턴 3)

```json
{
  "tool": "Write",
  "id": "tu-004",
  "input": {
    "file_path": "docs/wiki/react-query-vs-swr.md",
    "content": "..."
  }
}
```

**Write가 생성하는 파일:**
```markdown
# React Query vs SWR 비교

> 작성: 2026-04-09 / 출처: 공식 문서 + npmtrends

---

## 한 줄 요약

| | React Query | SWR |
|---|---|---|
| **만든 곳** | TanStack (커뮤니티) | Vercel |
| **철학** | 서버 상태 관리 전문 라이브러리 | 단순하고 가벼운 데이터 페칭 |
| **번들 크기** | 13.1 KB | 4.2 KB |
| **주간 다운로드** | 820만 | 310만 |

---

## 언제 뭘 쓸까

**React Query 선택 시**
- 복잡한 캐시 무효화 로직이 필요한 경우 (`invalidateQueries`)
- 낙관적 업데이트(optimistic update)가 중요한 경우
- 무한 스크롤 / 페이지네이션 (`useInfiniteQuery`)
- 강력한 DevTools로 캐시 상태를 눈으로 확인하고 싶을 때

**SWR 선택 시**
- Next.js 프로젝트 (Vercel 팀 작품, 궁합 좋음)
- 단순 GET 요청 위주, 복잡한 mutation 없을 때
- 번들 크기가 중요한 경우 (모바일 퍼스트 등)
- 빠르게 프로토타입 만들 때

---

## 주요 API 비교

### 기본 데이터 페칭

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 서버 데이터 변경

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## 결론

React Query가 기능이 더 풍부하고 생태계도 크지만,
단순한 프로젝트에선 SWR의 가벼움이 장점이 될 수 있다.

새 프로젝트 기준으로는 **React Query 권장**
(더 넓은 커뮤니티, 문서, DevTools 지원)

---

*참고: [TanStack Query 공식](https://tanstack.com/query) · [SWR 공식](https://swr.vercel.app) · [npmtrends](https://npmtrends.com)*
```

**권한 확인:**
```
┌─────────────────────────────────────────────────────┐
│  Create docs/wiki/react-query-vs-swr.md             │
│                                                     │
│  # React Query vs SWR 비교                          │
│  ...                                                │
│                                                     │
│  [y] 허용   [n] 거부                                 │
└─────────────────────────────────────────────────────┘
사용자: y
```

**[중간 결과 ④] 파일 생성 완료:**
```
{ filePath: "docs/wiki/react-query-vs-swr.md", bytesWritten: 2104 }
```

---

## STEP 6: 최종 응답 (턴 4)

```
docs/wiki/react-query-vs-swr.md 에 비교 문서를 저장했습니다.

조사한 내용:
  - React Query 공식 문서 (tanstack.com)
  - SWR 공식 문서 (swr.vercel.app)
  - npm 다운로드 트렌드 (npmtrends.com)

핵심 차이:
  React Query: 기능 풍부, 번들 13KB, 주 820만 다운로드
  SWR: 가볍고 단순, 번들 4KB, Next.js 친화적

새 프로젝트에는 React Query를 권장하지만,
Next.js + 단순 GET 위주 프로젝트라면 SWR도 충분합니다.
```

---

## 전체 타임라인

```
T+0.0s   "React Query vs SWR 조사해줘" 입력
T+0.1s   QueryEngine 진입
T+0.2s   [턴1] WebFetchTool × 2 병렬
           tanstack.com + swr.vercel.app 동시 요청
           URL 권한 확인 → 사용자 'a'
T+4.5s   권한 승인 후 실행
T+6.8s   두 사이트 fetch + HTML→Markdown 변환 완료  (2.3s)
T+7.0s   [턴2] WebFetchTool → npmtrends.com
           URL 권한 확인 → 사용자 y
T+10.5s  fetch + 트렌드 수치 수신               (3.5s)
T+10.7s  [턴3] Write → docs/wiki/react-query-vs-swr.md
           권한 확인 → 사용자 y
T+14.0s  파일 생성 완료
T+14.1s  [턴4] 최종 응답 스트리밍
T+14.5s  출력 완료

────────────────────────────────────────────────
총 소요:    ~14.5초 (사이트 응답 + 사용자 확인 포함)
API 호출:   4턴
도구 실행:  4회 (WebFetch×3, Write×1)
사용자 확인: 3회 (각 도메인 첫 접근 + Write)
생성 파일:  1개
총 비용:    ~$0.0011
────────────────────────────────────────────────
```

---

## 컴포넌트별 역할

```
┌───────────────────────────────────────────────────────────────┐
│  사용자: "React Query vs SWR 조사해서 위키에 저장해줘"         │
└────────────────────────┬──────────────────────────────────────┘
                         │
                  ┌──────▼──────┐
                  │  main.tsx   │  WebFetchTool 포함 도구 조립
                  └──────┬──────┘
                         │
                  ┌──────▼──────┐
                  │ QueryEngine │  세션 초기화
                  └──────┬──────┘
                         │
  ┌──────────────────────▼────────────────────────────────────┐
  │                    queryLoop                               │
  │                                                            │
  │  턴1  WebFetchTool × 2 병렬                               │
  │       tanstack.com → [사용자 'a'] → Markdown 요약 ①      │
  │       swr.vercel.app → (자동) → Markdown 요약 ②          │
  │                                                            │
  │  턴2  WebFetchTool                                        │
  │       npmtrends.com → [사용자 y] → 다운로드 수치 ③       │
  │                                                            │
  │  턴3  Write                                               │
  │       → [사용자 y] → react-query-vs-swr.md ④            │
  │                                                            │
  │  턴4  최종 응답   → 요약 + 파일 경로 안내                  │
  └───────────────────────────────────────────────────────────┘
                         │
               docs/wiki/react-query-vs-swr.md
```

---

## WebFetchTool이 다른 도구와 다른 점

```
FileReadTool:
  로컬 파일시스템에서 읽음
  경로 기반 접근 제어 (checkReadPermissionForTool)

WebFetchTool:
  외부 URL에서 읽음
  도메인 기반 접근 제어 (preapprovedHosts 목록)
  HTML → Markdown 자동 변환
  AI 프롬프트로 내용 필터링 가능 (applyPromptToMarkdown)
  최대 50,000자 제한 (긴 문서 자동 잘라냄)

권한 패턴:
  preapprovedHosts에 있으면 → 자동 허용
  없으면 → 도메인 단위로 사용자 확인
           'a' 선택 시 해당 도메인 이후 자동
```

## 컴포넌트가 주고받은 데이터 한눈에

| 턴 | 도구 | 입력 | 출력 | 비고 |
|----|------|------|------|------|
| 1 | WebFetchTool | tanstack.com URL + 요약 프롬프트 | 핵심 특징 마크다운 | **'a' 선택** |
| 1 | WebFetchTool | swr.vercel.app URL + 요약 프롬프트 | 핵심 특징 마크다운 | **병렬 실행**, 자동 |
| 2 | WebFetchTool | npmtrends.com URL | 다운로드 수치 텍스트 | 사용자 y |
| 3 | Write | 비교 마크다운 전체 (2,104 bytes) | 파일 생성 완료 | 사용자 y |
| 4 | (없음) | — | 최종 요약 텍스트 | 스트리밍 |
