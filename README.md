# 9주차 실습 2 · 검색·정렬·페이지네이션 게시판 개선하기

웹서비스설계 9주차 실습 예제(1,000건 인메모리 게시판)를 실행해 기능을 검증하고, 발견한 문제를 수정했습니다.
수정 전 원본은 첫 커밋(`ef7e383`)에 그대로 남겨 두어 **커밋 diff로 기존 코드와 비교**할 수 있습니다.

- 백엔드: Node.js + Express (`server.js`)
- 프론트엔드: 단일 HTML + 바닐라 JS (`index.html`)

---

## 1. 실행 방법

```bash
npm install          # package.json 기준으로 express 설치
node server.js       # 서버 실행
# 브라우저에서 http://localhost:3000 접속
```

- Node.js 18 이상 필요 (`Object.hasOwn` 사용). 개발 환경: Node.js v24.22.0, Express 5.x
- `node_modules`는 저장소에 포함하지 않았습니다(`.gitignore`).

## 2. 커밋 이력

| 커밋 | 내용 |
|---|---|
| `ef7e383` | 9주차 실습 원본 예제 코드 추가 (수정 전) |
| `00e515f` | 서버: 정렬 키 상속 속성 차단, 쿼리 타입 검증, 커서 API 검색 추가 |
| `5e80827` | 프론트: 커서 검색·정렬 비활성화, 오래된 응답 차단, 디바운스 중복 요청 제거, URL 상태 복원 보완 |
| `436fb56` | 주석: 코드 변경 내용에 대해 개선 항목별 정리 |

- 원본 → 최종 전체 비교: https://github.com/sjlys0205/26_02_webPractice02/compare/ef7e383...main
- 코드에서 수정한 위치는 `[정렬 보안]`, `[검색 기능 개선]`, `[커서 페이징]`, `[Race Condition]`, `[URL 상태 복원]` 등의 태그 주석으로 표시했습니다.

---

## 3. 발견한 문제와 수정 내용

### 3-1. 검색·정렬

| 문제 (수정 전) | 원인 | 수정 |
|---|---|---|
| `sort=__proto__`, `sort=hasOwnProperty` → **500 에러** / `sort=constructor` → 정렬이 아예 수행되지 않음 | `SORT_WHITELIST[key]`는 프로토타입 체인까지 조회하므로 `Object.prototype`의 상속 속성이 화이트리스트를 통과. `Object.freeze`는 값 변경만 막고 상속 조회는 막지 못함 | `resolveSortKey()`: `Object.hasOwn(SORT_WHITELIST, key)`로 **직접 가진 키(latest/oldest/views)만 허용**, 나머지는 `latest`. 실제 적용한 키를 `meta.sort`로 응답 |
| `search=a&search=b` → **500 에러** | 같은 이름이 반복되면 Express가 배열로 전달 → 배열에는 `.trim()`이 없음 | `toQueryString()`: 문자열이 아니면 빈 문자열로 처리 (search·sort 공용) |
| 커서 API에 검색 없음 | `/api/posts/cursor`가 `search`를 읽지 않음 | `matchesSearch()`로 검색 조건을 함수로 분리해 오프셋·커서 API가 같은 기준 사용 |

- 조회순 동점 처리: `views DESC, id DESC` 비교 함수(원본부터 존재)를 유지했습니다.

### 3-2. 페이지네이션

| 문제 (수정 전) | 수정 |
|---|---|
| 커서 API·커서 모드 화면에서 검색어 무시 | 서버: **검색 조건 → 커서 조건(`id < cursor`) → `id DESC` 정렬 → `limit + 1` 슬라이스** 순서로 처리. 프론트: 커서 요청에 `search` 전달 |
| 커서 모드에서도 정렬 선택이 활성화(효과 없음) | 커서 모드에서 정렬 선택 **비활성화** + "id DESC 고정" 안내 문구 표시, 응답 `meta.order: "id DESC"` |
| 검색어 변경 시 누적 목록 초기화가 응답 도착 후에 일어남 | `resetCursor()`로 **요청 전에** 커서·누적 목록 초기화 (검색, 조회, 모드 전환, URL 복원 시) |

### 3-3. 비동기 처리

| 문제 (수정 전) | 수정 |
|---|---|
| 입력 후 300ms 안에 [조회]를 누르면 요청 2회 | `applySearch()`에서 `clearTimeout(debounceTimer)` → 요청 1회 |
| 오래된 응답이 화면을 덮어쓸 여지 | 요청마다 번호(`requestId`)를 부여하고, 응답 도착 시 `requestId !== latestRequestId`이면 버림 (AbortController와 이중 방어) |
| 4xx/5xx 응답도 정상 결과처럼 처리 | `res.ok` 검사 후 에러 처리. 최신 요청이 실패한 경우에만 오류 문구 표시 |
| (부가) 공백만 바뀐 검색어도 요청 | trim한 검색어가 이전과 같으면 요청하지 않음. Enter 키로 즉시 조회 |

### 3-4. URL 상태 복원 (오프셋 모드)

| 문제 (수정 전) | 수정 |
|---|---|
| `?sort=hack` → 정렬 선택창이 빈칸 | `restoreFromUrl()`에서 `SORT_KEYS` 화이트리스트로 정제 |
| `?page=999` → 서버는 100페이지를 주는데 URL·state는 999 | 응답의 `meta.page`, `meta.sort`를 state·선택창에 반영하고 `syncUrl()`로 주소창 교정 |
| 요청 **전에** `pushState` → 취소된 요청도 기록에 남음 | 응답 **후** `syncUrl()`: 사용자 조작은 `pushState`, 새로고침·뒤로 가기 복원은 `replaceState`, 같은 주소는 중복 기록하지 않음 |
| 새로고침·뒤로 가기 코드가 중복, 커서 모드에서 뒤로 가기 시 화면 불일치 | `restoreFromUrl()` 하나로 통합, URL 복원 시 오프셋 모드로 전환 |

> 커서 모드는 URL에 상태를 저장하지 않습니다(가이드 기준: 오프셋 모드만 동기화). 커서 모드에서 새로고침하면 오프셋 모드로 복원됩니다.

---

## 4. 테스트 결과

캡처는 [`docs/screenshots`](docs/screenshots)에 있습니다. `B`는 수정 전(Before), `A`는 수정 후(After)입니다.

### 4-1. 수정 전 (원본 코드)

| 번호 | 입력 | 실제 결과 |
|---|---|---|
| B01 | `/api/posts?sort=__proto__` | 500, `TypeError: The comparison function must be either a function or undefined` |
| B02 | `/api/posts?sort=hasOwnProperty` | 500, `TypeError: Cannot convert undefined or null to object` |
| B03 | `/api/posts?search=a&search=b` | 500, `TypeError: (req.query.search \|\| "").trim is not a function` |
| B04 | `/api/posts/cursor?search=AbortController` | 검색 무시, #1000(B-Tree…)부터 전체 목록 |
| B05 | 커서 탭에서 `Redis` 검색 | 요청이 `cursor?limit=10`(검색어 없음), 정렬 선택 활성 상태 |
| B06 | `/?sort=hack` | 정렬 선택창 빈칸, 요청에 `sort=hack` 그대로 |
| B07 | `/?page=999` | 화면은 100페이지, 주소창 `page=999`, 검사기 SQL `OFFSET 9980` |
| B08 | 입력 후 300ms 안에 [조회] | `/api/posts` 요청 2회 |

### 4-2. 수정 후

| 번호 | 입력 | 기대 결과 | 실제 결과 |
|---|---|---|---|
| A01 | `/api/posts?sort=__proto__` | 200, latest | 200, `meta.sort: "latest"` |
| A02 | `/api/posts?sort=hasOwnProperty` | 200, latest | 200, `meta.sort: "latest"` |
| A02_2 | `/api/posts?sort=constructor` | 200, latest | 200, `meta.sort: "latest"` |
| A03 | `/api/posts?search=a&search=b` | 200, 검색어 무시 | 200, `totalCount: 1000` |
| A04 | `/api/posts/cursor?search=AbortController` | AbortController 글만, id DESC | #996, #984 … #888, `order: "id DESC"`, `hasNext: true`, `nextCursor: 888` |
| A05 | 커서 탭에서 `Redis` 검색 | 검색어 전달, 정렬 비활성화 | 요청 `cursor?limit=10&search=Redis`, Redis 글만 표시, 정렬 선택 비활성화 + 안내 문구 |
| A05_2 | A05에서 "더 보기"를 끝까지 | 누락·중복 없음, 끝에서 버튼 비활성화 | #991부터 #7까지 표시(Redis 글 83건), "더 이상 글이 없습니다" 버튼 비활성화 |
| A06 → A06_2 | `/?sort=hack` | latest로 교정 | 주소창 `sort=latest`, 선택창 "최신순" |
| A07 → A07_2 | `/?page=999` | 100페이지로 교정 | 주소창 `page=100`, 검사기 SQL `OFFSET 990` |
| A08 | 입력 후 300ms 안에 [조회] | 요청 1회 | 요청 1회 |
| A09 | `abort` 연속 타이핑 | 요청 1회 (디바운스) | 요청 1회 |
| A10 | 3G 쓰로틀링에서 검색어를 바꿔 입력 | 이전 요청 취소, 최종 결과만 표시 | 취소(canceled) 요청 확인, 콘솔에 AbortController 취소 로그, 최종 화면은 AbortController 결과 |
| A11_1 | `Redis` 검색 → 조회수순 → 3페이지 | URL·입력창·정렬·결과 일치 | `page=3&sort=views&search=Redis`, 조회수 내림차순, `OFFSET 20` |
| A11_2 | 뒤로 가기 1회 | 정렬 변경 직후 상태(1페이지) | `page=1&sort=views&search=Redis`, 선택창 "조회수순" |
| A11_3 | 뒤로 가기 1회 더 | 검색 직후 상태 | `page=1&sort=latest&search=Redis`, 선택창 "최신순" |
| A12 | A11_3에서 F5 | 같은 상태 유지 | 주소창·입력창·정렬·목록 동일 |
| A13_1-3 | `/api/posts?page=abc&limit=1000`, `page=0`, `page=-1` | page 1, limit 100으로 보정 | 세 경우 모두 `page: 1`, `limit: 100`, `totalPages: 10` |

### 4-3. 응답 시간 관찰 (`X-Response-Time`, 단위 ms, No throttling, 각 5회)

같은 게시글 #110-#101을 가져오는 두 요청과 첫 페이지를 비교했습니다. (`limit=10`이면 전체 100페이지이므로 100페이지 이상 스킵은 불가능해 90페이지로 측정)

| 요청 | 1 | 2 | 3 | 4 | 5 | 평균 |
|---|---|---|---|---|---|---|
| M01 오프셋 `page=1` | 0.13 | 0.17 | 0.10 | 0.15 | 0.09 | 0.13 |
| M02 오프셋 `page=90` (OFFSET 890) | 0.16 | 0.08 | 0.08 | 0.15 | 0.23 | 0.14 |
| M03 커서 `cursor=111` | 0.14 | 0.13 | 0.15 | 0.07 | 0.12 | 0.12 |

- 세 요청의 평균 차이(0.01-0.02ms)가 반복 측정의 흔들림보다 작아 **이 환경에서는 유의미한 차이가 관찰되지 않았습니다.**
- 이유: 예제는 DB가 아니라 메모리 배열 1,000건이고, 두 API 모두 매 요청마다 `filter`로 전체 배열을 훑기 때문에 작업량이 비슷합니다.
- 실제 DB에서는 `LIMIT 10 OFFSET 890`이 앞의 890행을 읽고 버리므로 뒤 페이지일수록 느려지고, 커서의 `WHERE id < 111 ORDER BY id DESC LIMIT 11`은 인덱스로 시작 위치를 바로 찾으므로 페이지 깊이와 관계없이 일정합니다. (DB 인덱스 성능 입증은 과제 범위 밖)
- M02_3은 브라우저 캐시로 304 응답이었으나, 서버는 요청을 처리한 뒤 응답하므로 측정값은 유효합니다.

---

## 5. 개념 설명

### ① 오프셋·커서 페이지네이션에 적합한 화면

| 방식 | 적합한 화면 | 이유 |
|---|---|---|
| 오프셋 (`LIMIT/OFFSET`, 페이지 번호) | 관리자 목록, 게시판처럼 **특정 페이지로 바로 이동**하거나 전체 페이지 수를 보여줘야 하는 화면 | `page`만 알면 어느 페이지든 바로 조회 가능, `totalPages` 계산 가능 |
| 커서 (`WHERE id < cursor`) | SNS 피드, 댓글, 채팅처럼 **무한 스크롤·"더 보기"**로 이어 읽는 화면 | 깊은 위치도 일정한 속도, 새 글이 추가돼도 목록이 밀려 **중복·누락이 생기지 않음** (대신 임의 페이지 이동·전체 페이지 수 표시는 어려움) |

### ② 디바운스와 요청 취소의 역할

- **디바운스(300ms)**: 입력이 멈춘 뒤 300ms가 지나야 요청을 보냅니다. 연속 타이핑 중의 **요청 개수 자체를 줄입니다** (요청을 보내기 전 단계).
- **요청 취소(AbortController)**: 이미 보낸 요청이 진행 중일 때 새 요청을 보내면 이전 요청을 취소합니다. 응답 순서가 뒤바뀌어 **오래된 결과가 화면을 덮어쓰는 문제(Race Condition)를 막습니다** (요청을 보낸 후 단계).
- 둘은 대체 관계가 아니라 보완 관계입니다. 디바운스로도 300ms 이상 간격의 요청은 겹칠 수 있으므로 취소가 필요하고, 본 코드는 여기에 요청 번호 비교를 더해 취소 직전에 도착한 응답까지 버립니다.

### ③ `limit + 1` 원리

10개를 보여줄 때 11개를 조회합니다. 11번째가 있으면 `hasNext = true`, 없으면 `false`이고, 화면에는 앞의 10개만 보냅니다. **`COUNT(*)`로 전체 개수를 세지 않고도** 다음 페이지 존재 여부를 알 수 있습니다.
단, 검색·커서 조건을 **모두 적용한 뒤에** 잘라야 합니다. 먼저 자르면 검색 결과가 더 있어도 `hasNext=false`가 될 수 있습니다.

### ④ offset·totalPages 계산식

```
offset     = (page - 1) × limit
totalPages = max(1, ceil(totalCount ÷ limit))
page       = min(요청 page, totalPages)   // 범위를 넘는 page는 마지막 페이지로 보정
```

- 예: 전체 1,000건, `limit=10` → `totalPages = 100`, `page=90` → `offset = 890`
- 예: `Redis` 검색 83건, `limit=10` → `totalPages = ceil(8.3) = 9`
- 결과가 0건이어도 `max(1, …)`로 `totalPages = 1`이 되어 빈 페이지가 정상 응답됩니다.
- 입력 방어: `page=0, -1, abc` → 1, `limit=1000` → 100(상한)

---

## 6. AI 활용 및 직접 검증

**사용 도구:** Claude Code

| AI가 수행한 부분 | 직접 수행·검증한 부분 |
|---|---|
| 예제 코드 분석, 버그 원인 분석 | 원본·수정본 실행, 모든 테스트 직접 수행 및 캡처(B01-B08, A01-A13, M01-M03) |
| `server.js`, `index.html` 수정 코드 초안 작성 | 수정 코드를 읽고 이해한 뒤 주석 직접 작성 |
| 테스트 시나리오 제안 및 검토 | 테스트 결과 판단 및 커밋 결정 |
| README 초안 작성 | README 내용 수정 및 검토 |
