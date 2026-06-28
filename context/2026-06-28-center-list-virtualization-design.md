# 센터 리스트 개선 — 정확한 카운트 · 가상 스크롤 · 정확한 뒤로가기 복원

작성일: 2026-06-28

## 배경 / 문제

모든 외부 센터 리스트 페이지(loveydovey, melting, tingle, babechat, tikita, rofan, zeta, whif, chub)는
`lib/centerCounts`, `lib/useInfiniteScroll`, `lib/useScrollRestore`를 공유하는 동일 패턴으로 동작한다.
loveydovey(`app/(loveydovey)/loveydovey/page.tsx`)를 기준으로 확인된 현재 구조의 결함:

1. **카운트가 부분 로드된 배열에서 계산됨.** `counts = viewCounts(chars)` 가 60개씩 fetch한
   `chars`(현재까지 로드분)에서 계산되어, 진행중/대기/완결 숫자가 실제 전체 총합이 아니라
   스크롤할수록 늘어난다. 사실상 "렌더된 카드 수"를 세는 셈.
2. **필터/정렬이 전부 부분 배열 대상 클라이언트 처리.** 뷰 탭·태그·성별·검색·정렬이
   로드된 분량만 본다. 예: 완결 탭은 로드 윈도우 밖의 완결 항목을 못 보여준다.
3. **두 겹의 페이지네이션이 충돌.** API fetch 페이지(`FETCH_SIZE=60`, `loadMore`)와
   별도 클라 노출 카운터(`useInfiniteScroll` 30 시작, `slice(0, count)`)가 같은 sentinel
   교차에서 동시에 동작. append → 전체 재정렬 → 재slice 로 "새 항목이 위로 튀는" 글리치 발생.
4. **뒤로가기 복원 불가.** `useScrollRestore`가 `scrollTop`을 저장하지만, 복귀 시 컴포넌트가
   재마운트되어 `chars`는 60개만 다시 fetch하고 `count`는 30으로 리셋 → 깊은 스크롤 위치를
   복원할 리스트 자체가 없다.

서버(`/api/collections`)는 이미 항목별 `completed`/`started`/`lastActivityAt`를 계산해 반환하므로,
정확한 뷰별 카운트는 달성 가능하다.

## 결정된 방향 (브레인스토밍 합의)

- **데이터 규모**: 한 센터당 수천(1천~5천) 개 가능 → 경량 인덱스를 **한 번에 전체 로드**하고
  필터/정렬/카운트는 클라이언트에서 처리.
- **작업 범위**: loveydovey 한 센터를 프로토타입으로 먼저 적용·검증 후 나머지 센터로 전파.
- **렌더링**: 가상 스크롤(가상화).
- **카운트 기준**: 진행/대기/완결 탭 옆 숫자는 **항상 전체 총합**(검색·태그·성별 필터와 무관).
  태그칩/성별칩 카운트는 기존대로 필터 반영 유지.
- **카드 높이**: 가상화를 위해 **고정 높이(균일 그리드)** — 제목 1줄·설명 2줄 고정·태그 1줄.
  측정형 가상화 라이브러리 의존성 추가하지 않음.

## 설계

### 1. 데이터 흐름 (핵심 수정)

60개씩 fetch + 부분 배열 처리 → **경량 인덱스 전체 1회 로드 후 클라 전처리**로 교체.

- **신규 API 모드**: `GET /api/collections?isLoveydovey=true&fields=index`
  - 해당 센터의 **전체** 항목 반환(`limit`/`offset` 무시).
  - 경량 필드만 반환:
    `id, title, coverImageUrl, description, tags, createdAt, lastActivityAt, completed, started, characters:[{ id, name, avatarUrl, gender }]`
  - 리스트에 불필요한 무거운 부분은 **생략**: lorebook 쿼리, `openingMessages`,
    `zetaMeta`/`meltingMeta`/`tikitaMeta`.
  - 기존 대화 집계(`completed`/`started`/`lastActivityAt`)는 유지(단일 grouped 쿼리)하여
    상태 플래그는 서버 진실값 사용.
  - 약 5천 항목도 gzip 후 수백 KB 수준 — 단일 fetch 후 캐시.
- 기존 `fields=basic`(드롭다운용 id/title/sourceUrl) 모드는 그대로 유지. `index`는 별도 모드.

### 2. 카운트

`counts = 전체 인덱스 배열 reduce`(`viewCounts`) — **렌더링과 완전 분리**된 순수 숫자 집계.
항상 전체 총합. 태그칩/성별 카운트는 기존 필터 반영 로직 유지.

### 3. 필터 / 정렬

전부 클라이언트, 전체 in-memory 배열 대상(로직은 `centerCounts`/`listSort`/`tagGroups` 재사용).
배열이 완전하므로 결과가 정확해진다. 뷰 탭·태그·성별·검색·정렬·랜덤 모두 전체 집합 대상.

### 4. 렌더링 — 가상 스크롤

- 두 겹 페이지네이션 제거: `FETCH_SIZE=60` fetch + `useInfiniteScroll` 노출 카운터 삭제.
  "새 항목이 위로 튀는" 글리치는 append+재정렬+재slice 에서 비롯되므로 증분 fetch 제거로 해소.
- **신규 `VirtualCardGrid` 컴포넌트**: 의존성 없는 2열 윈도잉 그리드.
  스크롤 컨테이너의 `scrollTop`에서 보이는 행(+overscan)만 렌더하고, 상/하 스페이서로 총 높이 확보.
  - props: `items`, `renderItem(item) => ReactNode`, `columns`(기본 2), `rowHeight`, `gap`, `overscan`.
  - 스크롤 컨테이너 ref를 받아 `scroll` 이벤트로 가시 행 범위 계산.
- **균일 행 높이 필요** → 카드 고정 높이.
  - 각 센터 카드 마크업/CSS 클래스는 그대로 두되, 카드 높이만 고정:
    제목 1줄(`-webkit-line-clamp:1`), 설명 2줄 고정 박스, 태그 1줄.
  - loveydovey의 경우 `.lovey-card`에 고정 높이 적용. 정확한 px 값은 구현 시 실측으로 확정.

### 5. 정확한 뒤로가기 복원

- **모듈 레벨 캐시**(센터 키별): 전체 인덱스 배열 보관. 클라이언트 네비게이션 간 생존(JS 모듈 유지),
  하드 리로드 시에만 소실 → 재fetch.
- **sessionStorage** UI 상태: `view, sort, query, selectedTags, genderFilter, randomSeed, searchOpen, scrollTop`.
- 캐릭터 상세 → 뒤로가기 복귀 시: 캐시에서 즉시 hydrate(스피너 없음) → 필터 재적용 →
  `scrollTop` 설정 → 가상화가 정확한 가시 윈도우 재구성.
  전체 항목이 메모리에 있으므로 "복원할 리스트가 짧아서 못 돌아가는" 문제 없음.
- (선택) 배경 stale-while-revalidate 재검증: 스크롤/상태 방해 없이 인덱스 갱신.

### 6. 공용 레이어 (프로토타입 → 전파)

- §1~§5를 **`useCenterList(centerParam, storagePrefix)`** 훅으로 묶음.
  - 반환: `{ items, counts, visibleChars, tagGroups, tCounts, genderBuckets,
    view, setView, sort, setSort(handleSort), query, setQuery, selectedTags, toggleTag,
    genderFilter, setGenderFilter, searchOpen, toggleSearch, randomSeed, loading, scrollRef, ... }`
  - 인덱스 fetch + 모듈 캐시 + sessionStorage 상태 + 파생값 계산을 캡슐화.
  - 기존 `useInfiniteScroll` / `useScrollRestore` / 수동 fetch·loadMore 를 대체.
- loveydovey가 먼저 `useCenterList` + `VirtualCardGrid` 채택.
- 나머지 8개 센터는 데이터/스크롤 배선만 훅으로 교체하고 각자 카드 마크업은 유지하며 순차 이주.

## 영향 범위 / 파일

- 신규: `lib/useCenterList.ts`, `components/ui/VirtualCardGrid.tsx`
- 수정: `app/api/collections/route.ts` (`fields=index` 모드 추가)
- 수정: `app/(loveydovey)/loveydovey/page.tsx` (훅 + 가상 그리드로 전환)
- 수정: `app/globals.css` (`.lovey-card` 고정 높이; 추후 센터별 동일 처리)
- 정리 대상(전파 후): `lib/useInfiniteScroll.ts`, `lib/useScrollRestore.ts` (훅에 흡수되면 제거 검토)
- 가이드 동기화: 사용자 체감 변화(가상 스크롤·정확 카운트) 발생 시
  `app/(main)/guide/page.tsx` 및 `context/adding-a-center.md` 갱신 검토.

## 비목표 (YAGNI)

- 서버 사이드 필터/정렬/페이지네이션(수천 규모에선 불필요).
- 측정형 가상화 라이브러리 도입(고정 높이로 회피).
- 한 번에 9개 센터 전체 이주(프로토타입 검증 후 전파).

## 검증 기준

- 진행/대기/완결 카운트가 스크롤·필터와 무관하게 DB 전체 총합과 일치.
- 수천 개 항목에서 스크롤이 매끄럽고 새 항목이 위로 튀지 않음.
- 캐릭터 상세 진입 후 뒤로가기 시 직전 스크롤 위치·필터·뷰 그대로 복원.
- 태그/성별/검색/정렬(랜덤 포함)이 전체 집합 기준으로 정확.
