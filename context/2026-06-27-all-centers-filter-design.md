# 전체 센터 페이지 검색·필터·태그 개선 설계

작성일: 2026-06-27
대상: `apps/web/app/(main)/explore/all/page.tsx` (+ `apps/web/app/globals.css`)

## 배경 / 문제

전체 센터(`/explore/all`) 페이지는 **텍스트 검색 + 센터 칩 + 정렬/뷰탭**만 제공한다.
개별 센터 페이지(zeta·melting 등)에 있는 다음 기능이 빠져 있다:

- 카테고리별 **태그 필터바**(그룹 + 카운트)
- **성별 필터**(남/여/멀티/미분류)
- 검색이 **제목·태그**까지만 매칭(설명·캐릭터명 누락)

## 목표 (사용자 확정)

1. 태그 필터 추가 — 개별 센터와 동일한 `TagFilterBar`(카테고리 그룹·카운트·AND 선택)
2. 검색 범위 확장 — 제목·태그 → **제목·태그·설명·캐릭터명**
3. 성별 필터 추가

제외(YAGNI): 세계관/캐릭터 구분 필터. (카드의 세계관/캐릭터 뱃지 표시는 유지)

## 접근 (A안: 기존 인프라 이식)

이미 검증된 공용 모듈을 전체 센터 페이지에 재사용한다. API 변경 불필요 —
`/api/collections?all=true`는 이미 `characters[].gender`와 `tags`를 내려준다.

| 재사용 모듈 | 용도 |
|------------|------|
| `components/ui/TagFilterBar` | 카테고리별 태그 칩 + 카운트 렌더 |
| `lib/tagGroups` (`buildTagGroups`, `CenterTagConfig`) | 전역 `CenterTag` 분류로 그룹화 |
| `lib/centerCounts` (`tagCounts`) | 태그별 등장 수 |
| `lib/cardGender` (`cardGenderBucket`, `availableGenderBuckets`) | 카드(컬렉션) 단위 성별 버킷 |
| `GET /api/center-tags` | 전역 태그 설정(`tags`, `categories`) |

전역 태그(`CenterTag`)는 센터 무관 단일 분류이므로 전체 센터에 그대로 적용된다.

## 변경 상세

### 1. 검색 범위 확장
`matchesQuery(c)`를 제목·태그 외에 `description`, `characters[].name`까지 부분일치하도록 확장.
(검색은 raw 값 대상 — 화면 렌더가 아니므로 placeholder 치환 불필요.)

### 2. 성별 필터
- 상태: `genderFilter: GenderBucket | 'all'`
- `Col.characters`에 `gender?: string | null` 타입 추가(데이터는 이미 존재).
- `availableGenderBuckets(set)`로 실제 존재하는 버킷만 카운트와 함께 칩 노출.
- 판정: `cardGenderBucket(col.characters)` 재사용.

### 3. 태그 필터바
- 마운트 시 `/api/center-tags` → `tagConfig` 상태.
- `buildTagGroups(set.flatMap(c => c.tags), tagConfig)` + `tagCounts(set)`.
- 상태: `selectedTags: string[]` — 선택 태그를 **모두** 가진 카드만 통과(AND).

### 필터 순서 / 카운트 기준
적용 순서: 뷰탭 → 센터칩 → 성별 → 검색어 → 태그.
성별·태그의 **노출 옵션과 카운트는 자기 자신을 제외한 나머지 필터가 적용된 집합** 기준으로
계산(좁혀가기 UX) — 개별 센터/캐릭터 페이지와 동일.

구현상: `base = view+center 적용` → 성별 옵션은 `base+query` 기준,
태그 옵션·카운트는 `base+gender+query` 기준, 최종 `filtered`는 전부 적용.

### UI 배치
기존 `🔍 검색` 패널 내부, 위→아래: 텍스트 입력 → 센터 칩(기존) → 성별 칩(신규) → 태그 필터바(신규).
검색 닫기(`toggleSearch`) 시 `query`·`selectedCenters`·`genderFilter`·`selectedTags` 모두 초기화.

### CSS
전역 재사용 칩 클래스 `.chip` 추가(`app/globals.css`) — `TagFilterBar`의 `chipClass`로 사용,
`accentVar="--accent"`. 성별/센터 칩도 점진적으로 통일 가능하나 이번엔 태그바에만 적용.

## 리스크 / 비고
- 렌더 윈도잉(`useInfiniteScroll` + `slice(0,count)`)·이미지 lazy 로딩은 이미 적용됨 — 유지.
- 데이터는 클라이언트에서 전량 로드 후 필터(기존 패턴 유지). 서버 필터로의 이동은 범위 외.
