# 시나리오: 성능 분석 리포트 (BashTool 진단 활용)

> **출처 안내 (2026-08-20 이관)**
>
> 이 문서는 2026-04에 작성한 **본인의 아키텍처 분석 기록**입니다.
> 원래 참조했던 소스 트리는 자체 LICENSE에 *"leaked proprietary source code
> belonging to Anthropic, PBC / NOT FOR REDISTRIBUTION"* 이라고 명시돼 있어
> 저장소에서 삭제했고, **이 문서에 있던 원문 코드 발췌(총 4개 블록)도 함께 제거**했습니다.
>
> 남은 것은 전부 본인이 쓴 산문 분석과 직접 그린 구조도입니다.
> 공식 문서: https://docs.claude.com/en/docs/claude-code
>
> Status: finished · Written: 2026-04-08 · Sanitized: 2026-08-20

---

> **핵심 컴포넌트**: BashTool — 실행 도구가 아닌 **진단 도구**로 사용
> 이 시나리오에서만 등장하는 흐름: 측정 → 결과 파싱 → 코드 확인 → 원인 특정 → 보고서 작성

---

## 시나리오 설정

> **사용자 입력**: `"API 응답이 요즘 너무 느려. 어디가 병목인지 찾아줘"`

코드를 고치는 것이 목적이 아니라 **어디가 문제인지 찾아서 보고서로 정리**하는 작업.
BashTool이 실행 도구(코드 빌드, 파일 생성)가 아닌 **측정·분석 도구**로 쓰이는 패턴이다.

```
통과 순서:
BashTool(엔드포인트 응답시간 측정) → BashTool(프로파일러 실행) →
FileReadTool(프로파일 결과 읽기) → GrepTool(코드 확인) →
FileReadTool(의심 함수 읽기) → Write(분석 보고서 생성) → 최종 응답
```

---

## STEP 0: 앱 시작 (main.tsx)

```
git status
└─> "On branch main  /  M src/api/userRouter.ts"
    (최근 수정된 파일이 있음 — 단서가 될 수 있음)

도구 풀 조립
└─> [GlobTool, GrepTool, FileReadTool, BashTool, Write, ...]  총 12개
```

---

## STEP 1: 사용자 입력 접수 (QueryEngine)

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

---

## STEP 2: 첫 번째 API 호출 (턴 1)

**Claude 내부 thinking:**
```
"느리다는 게 어느 엔드포인트인지, 얼마나 느린지 수치를 먼저 재야 한다.
 서버가 로컬에 떠 있다면 curl로 측정할 수 있다.
 먼저 서버가 실행 중인지, 주요 엔드포인트가 뭔지 확인하자."
```

**결정한 도구 (2개 병렬):**
```json
[
  {
    "tool": "Bash",
    "id": "tu-001",
    "input": { "command": "curl -s -o /dev/null -w '%{time_total}\\n' http://localhost:3000/api/users 2>&1" }
  },
  {
    "tool": "Bash",
    "id": "tu-002",
    "input": { "command": "curl -s -o /dev/null -w '%{time_total}\\n' http://localhost:3000/api/products 2>&1" }
  }
]
```

**BashTool 권한 확인:**
```
1단계: AST 분석 → curl, /dev/null 사용 — 출력 리다이렉션 있음 → 검사
       /dev/null은 차단 목록에 없음 (읽기가 아닌 쓰기 방향, 안전)
2단계: 규칙 매칭 → "curl:*" 규칙 없음
3단계: 사용자 확인
  → "curl -s -o /dev/null ... 실행하시겠습니까? [y/n]"
  사용자: y (두 명령 동시 승인)
```

**StreamingToolExecutor: 2개 BashTool 병렬 실행**

**[중간 결과 ①] 응답 시간 측정:**
```
tu-001 (/api/users):    2.847   ← 2.8초! 심각
tu-002 (/api/products): 0.124   ← 0.1초, 정상
returnCode: 0 (둘 다)
```

**Claude의 판단:**
```
"명확하다. /api/users가 2.8초, /api/products는 0.1초.
 users 엔드포인트에 병목이 있다.
 git status에서도 userRouter.ts가 최근 수정됐다고 했으니
 거기가 원인일 가능성이 높다."
```

---

## STEP 3: BashTool — Node.js 프로파일러 실행 (턴 2)

**도구 호출:**
```json
{
  "tool": "Bash",
  "id": "tu-003",
  "input": {
    "command": "node --prof --prof-process src/scripts/bench-users.js 2>&1 | head -50"
  }
}
```

**권한 확인:**
```
1단계: AST 분석 → node --prof → 프로파일링 플래그, 안전
2단계: 규칙 매칭 → 없음
3단계: 사용자 확인 → y
```

**[중간 결과 ②] 프로파일 결과 (상위 50줄):**
```
Statistical profiling result, total: 8432 ticks

 [Bottom up (heavy) profile]:
  ticks  parent  name
  5821   69.0%   node:internal/crypto/hash
  5821  100.0%   /app/src/api/userRouter.ts:hashPassword
  5821  100.0%   /app/src/api/userRouter.ts:getUsers
   412    4.9%   node:net
   301    3.6%   /app/src/db/userRepository.ts:findAll
   189    2.2%   JSON.stringify
```

**Claude가 병목 특정:**
```
"원인 발견! 전체 실행 시간의 69%가 crypto/hash에서 소비되고 있다.
 getUsers 함수 안에서 hashPassword를 부르고 있는데,
 목록 조회 시 비밀번호를 해싱할 이유가 없다.
 코드를 직접 확인해서 왜 그런지 보자."
```

---

## STEP 4: GrepTool — 의심 함수 위치 확인 (턴 3)

```json
{
  "tool": "Grep",
  "id": "tu-004",
  "input": {
    "pattern": "hashPassword",
    "path": "src",
    "output_mode": "content",
    "-n": true
  }
}
```

**권한 확인:**
```
GrepTool.isReadOnly() = true → 자동 허용
```

**[중간 결과 ③] hashPassword 호출 위치:**
```
src/api/userRouter.ts:31:  const hashed = await hashPassword(user.password)
src/api/userRouter.ts:58:  users = users.map(u => ({ ...u, password: hashPassword(u.password) }))
src/utils/auth.ts:12:  export async function hashPassword(plain: string): Promise<string> {
```

**Claude:**
```
"58번 줄! getUsers 안에서 조회한 모든 유저의 비밀번호를
 다시 해싱하고 있다. 응답에 해시값을 포함시키려는 의도인 듯한데,
 이건 완전히 불필요한 작업이다.
 userRouter.ts를 직접 읽어서 정확한 맥락을 파악하자."
```

---

## STEP 5: FileReadTool — 문제 코드 확인 (턴 4)

```json
{
  "tool": "Read",
  "id": "tu-005",
  "input": {
    "file_path": "src/api/userRouter.ts",
    "offset": 50,
    "limit": 20
  }
}
```

**[중간 결과 ④] 문제가 있는 코드 부분:**
> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

**Claude 최종 분석:**
```
"원인 완전히 파악됨.

 목록 조회 시 비밀번호를 해싱하려는 의도였는데,
 진짜 해결책은 응답에서 password 필드 자체를 제거하는 것.
 hashPassword는 bcrypt 기반이라 의도적으로 느리게 설계된 함수인데,
 유저 수만큼 반복 호출하니 2.8초가 나온 것.

 수정 방향: users.map 안에서 password를 삭제 ({ ...u, password: undefined })
 이 분석을 보고서로 정리하자."
```

---

## STEP 6: Write — 분석 보고서 생성 (턴 5)

```json
{
  "tool": "Write",
  "id": "tu-006",
  "input": {
    "file_path": "docs/perf-report-2026-04-09.md",
    "content": "..."
  }
}
```

**Write가 생성하는 파일:**
```markdown
# API 성능 분석 보고서

**날짜**: 2026-04-09
**대상**: GET /api/users

---

## 요약

| 엔드포인트 | 응답시간 | 상태 |
|------------|---------|------|
| GET /api/users | **2,847ms** | 🔴 심각 |
| GET /api/products | 124ms | 🟢 정상 |

---

## 원인 분석

### 프로파일링 결과

```
전체 실행 시간 중 69%가 crypto/hash 에서 소비됨
  hashPassword() ← getUsers() ← GET /api/users
```

### 문제 코드

`src/api/userRouter.ts:56-59`

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

### 근본 원인

`hashPassword`는 bcrypt 기반으로 **보안을 위해 의도적으로 느리게 설계**된 함수.
유저 100명 기준 bcrypt cost=10이면 단순 계산으로도 약 3초 소요.

목록 조회 응답에서 비밀번호를 숨기려는 의도였으나,
해싱이 아닌 **필드 제거**가 올바른 방법.

---

## 권장 수정

> _[원문 코드 발췌 제거 — 위 출처 안내 참조]_

예상 개선 효과: 2,847ms → **~50ms** (DB 조회 시간만 남음)

---

## 추가 확인 필요 사항

- [ ] `userRepository.findAll()`이 SELECT * 를 쓰는지 확인
      (password 컬럼을 DB 쿼리 단계에서 제외하는 게 더 안전)
- [ ] 다른 엔드포인트에도 같은 패턴이 있는지 검토
      (`grep -r "hashPassword" src/api/` 로 확인)
- [ ] 수정 후 /api/users 응답에 password 필드가 노출되지 않는지 검증

---

*분석 도구: node --prof, curl, Claude Code*
```

**권한 확인:**
```
┌───────────────────────────────────────────────────┐
│  Create docs/perf-report-2026-04-09.md            │
│  [y] 허용   [n] 거부                               │
└───────────────────────────────────────────────────┘
사용자: y
```

**[중간 결과 ⑤] 보고서 생성 완료:**
```
{ filePath: "docs/perf-report-2026-04-09.md", bytesWritten: 1876 }
```

---

## STEP 7: 최종 응답 (턴 6)

```
원인을 찾았습니다. docs/perf-report-2026-04-09.md 에 상세 분석을 저장했습니다.

문제:
  GET /api/users가 2.8초 걸리는 이유는
  src/api/userRouter.ts:58 에서 모든 유저의 비밀번호를
  응답 전에 bcrypt로 다시 해싱하고 있기 때문입니다.

  bcrypt는 보안을 위해 의도적으로 느린 함수인데,
  유저 수만큼 반복 호출하니 병목이 생긴 것입니다.

수정 방법 (한 줄):
  users = users.map(({ password, ...rest }) => rest)
  (비밀번호 필드 자체를 응답에서 제거)

예상 개선:  2,847ms → ~50ms
```

---

## 전체 타임라인

```
T+0.0s   "API가 느려. 어디가 병목인지 찾아줘" 입력
T+0.1s   QueryEngine 진입
T+0.2s   [턴1] BashTool × 2 병렬
           /api/users + /api/products 응답시간 측정
           권한 확인 → 사용자 y
T+5.0s   측정 완료 (2개 curl 병렬, 2.8s 기다림)
           /api/users: 2.847s 🔴, /api/products: 0.124s 🟢
T+5.2s   [턴2] BashTool → node --prof 프로파일러 실행
           권한 확인 → 사용자 y
T+10.5s  프로파일 결과 수신 (실행 5.3s)
           → crypto/hash 69% 점유 확인
T+10.7s  [턴3] GrepTool → hashPassword 위치 확인  (자동, 44ms)
T+10.8s  [턴4] FileReadTool → userRouter.ts:50-70 읽기  (자동, 11ms)
           → 문제 줄 특정: 58번 줄
T+11.1s  [턴5] Write → 분석 보고서 생성
           권한 확인 → 사용자 y
T+14.5s  보고서 생성 완료
T+14.6s  [턴6] 최종 응답 스트리밍
T+15.0s  출력 완료

────────────────────────────────────────────────────
총 소요:    ~15.0초 (실제 실행 대기 ~8초 + 사용자 확인 ~5초)
API 호출:   6턴
도구 실행:  6회 (Bash×3, Grep×1, Read×1, Write×1)
사용자 확인: 3회 (Bash 3회 — 실행마다 다른 명령이라 매번 확인)
생성 파일:  1개 (분석 보고서)
총 비용:    ~$0.0010
────────────────────────────────────────────────────
```

---

## 컴포넌트별 역할

```
┌─────────────────────────────────────────────────────────────────┐
│  사용자: "API가 느려. 어디가 병목인지 찾아줘"                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                     ┌──────▼──────┐
                     │  main.tsx   │  BashTool, GrepTool 등 조립
                     └──────┬──────┘
                            │
                     ┌──────▼──────┐
                     │ QueryEngine │  세션 초기화
                     └──────┬──────┘
                            │
  ┌─────────────────────────▼───────────────────────────────────┐
  │                      queryLoop                               │
  │                                                              │
  │  턴1  BashTool × 2 병렬  → 응답시간 측정 ①                 │
  │       [y] curl /users: 2.847s  /products: 0.124s            │
  │                                                              │
  │  턴2  BashTool            → node --prof 프로파일 ②          │
  │       [y] crypto/hash 69% 점유 확인                         │
  │                                                              │
  │  턴3  GrepTool            → hashPassword 위치 탐색 ③       │
  │       (자동) src/api/userRouter.ts:58                       │
  │                                                              │
  │  턴4  FileReadTool        → 문제 코드 확인 ④                │
  │       (자동) 58번 줄: 반복 bcrypt 해싱 발견                  │
  │                                                              │
  │  턴5  Write               → [y] 분석 보고서 생성 ⑤         │
  │                                                              │
  │  턴6  최종 응답           → 원인 + 수정 방법 출력            │
  └──────────────────────────────────────────────────────────────┘
                            │
                  docs/perf-report-2026-04-09.md
```

---

## BashTool: 실행 도구 vs 진단 도구

이 시나리오에서 BashTool이 쓰이는 방식은 다른 시나리오와 다르다.

```
다른 시나리오에서의 BashTool:
  "npx tsc --noEmit"    → 코드 유효성 검증 (O/X 판정)
  "python3 script.py"   → 실제 데이터 처리 실행
  "git commit"          → 파일 저장, 상태 변경

이 시나리오에서의 BashTool:
  "curl -w '%{time_total}'"  → 응답 시간 수치 획득
  "node --prof"              → 함수별 CPU 점유율 측정
  → 결과 자체가 분석 재료
  → 부수 효과 없음, 순수 측정

공통점: 둘 다 BashTool.call()을 통해 실행되지만
       Claude가 결과를 어떻게 해석하고 다음 행동을 결정하는지가 다름
```

## 이 시나리오의 핵심 추론 체인

```
curl 측정    → "users만 2.8초" (다른 엔드포인트와 비교해서 이상함)
     ↓
node --prof  → "crypto/hash 69%" (CPU가 어디서 쓰이는지)
     ↓
GrepTool     → "hashPassword가 getUsers 안에서 호출됨"
     ↓
FileReadTool → "map으로 모든 유저 반복 해싱 중"
     ↓
결론         → "bcrypt를 응답 생성 시마다 반복 호출하는 것이 원인"

각 단계가 이전 단계의 결과를 좁혀가면서
Claude가 스스로 가설을 세우고 검증하는 흐름
```

## 컴포넌트가 주고받은 데이터 한눈에

| 턴 | 도구 | 입력 | 출력 | 비고 |
|----|------|------|------|------|
| 1 | BashTool ×2 | `curl -w '%{time_total}'` × 2 엔드포인트 | 2.847s / 0.124s | **사용자 y**, 병렬 |
| 2 | BashTool | `node --prof ... \| head -50` | `crypto/hash 69%` | **사용자 y** |
| 3 | GrepTool | `hashPassword` in src | `userRouter.ts:58` | 자동 허용 |
| 4 | FileReadTool | `userRouter.ts` offset=50 limit=20 | 58번 줄 코드 | 자동 허용 |
| 5 | Write | 분석 보고서 마크다운 | `bytesWritten: 1876` | **사용자 y** |
| 6 | (없음) | — | 원인 + 수정 방법 텍스트 | 스트리밍 |
