# 센터 태그 필터 접기 + 뷰연동 설계

작성일: 2026-06-27
대상: `components/ui/TagFilterBar.tsx`, 9개 센터 리스트 페이지 + `explore/all`, `context/adding-a-center.md`, `CLAUDE.md`

## 배경 / 문제

1. **태그 필터바가 너무 길다** — `TagFilterBar`는 `maxHeight 220 + overflow auto`만 있고 접기가 없어 화면을 많이 차지한다. 센터별 페이지와 전체 센터 페이지 양쪽 모두.
2. **센터 페이지의 태그가 뷰탭과 무관** — 9개 센터는 `tagGroups = buildTagGroups(<전체 리스트>.flatMap(tags))`, `tCounts = tagCounts(<전체 리스트>)`로 계산해, 진행중 탭이어도 완결 카드의 태그까지 전부 노출된다. (전체 센터 `explore/all`은 이미 뷰+성별+검색 연동으로 구현돼 있음.)
3. **센터 추가 가이드(`context/adding-a-center.md`)에 검색/필터/태그 패턴이 없다** — 새 센터 추가 시 이 동작이 누락되기 쉽다.

## 목표 (사용자 확정)

1. `TagFilterBar` 접기 — **기본 접힘 + 마지막 상태 기억(센터별 localStorage)**.
2. 센터 페이지 태그 필터를 **뷰+성별+검색 모두 연동**(전체 센터와 동일한 좁혀가기). 진행중 탭이면 진행중 카드 태그만, 완결 탭이면 완결 카드 태그만.
3. `adding-a-center.md` 갱신 + `CLAUDE.md`에 "센터 공통 동작 변경 시 가이드 갱신" 규칙 추가.

## 설계 (A안: 공용 컴포넌트 + 페이지별 base)

### 1. TagFilterBar 접기 (`components/ui/TagFilterBar.tsx`)

- 새 옵션 prop `storageKey?: string`.
- 내부 상태 `collapsed: boolean` — `storageKey`가 있으면 마운트 시 `localStorage.getItem(storageKey)`로 복원, **기본값 `true`(접힘)**. 토글 시 `localStorage.setItem`.
- 항상 렌더되는 **헤더 토글 버튼**(`chipClass` 사용): `🏷 태그` + 선택 수 배지(`selected.length > 0`일 때 `(N)`) + `▾`(접힘)/`▴`(펼침).
- `collapsed === true`면 그룹 목록을 숨기고 헤더만. `false`면 기존 그룹/카운트/`✕ 전체 해제` 표시.
- `groups.length === 0`이면 종전대로 `null` 반환(헤더도 숨김).
- 기존 호출부 호환: `storageKey` 미전달 시 기본 접힘 + 비지속(상태 in-memory). (모든 소비처에 `storageKey`를 전달할 것.)

### 2. 센터 페이지 뷰연동 태그 base (9개 + explore/all)

각 페이지에서 `tagGroups`/`tCounts`의 입력을 **전체 리스트 → 뷰+성별+검색이 적용된 base(단, 태그 필터 자신은 제외)** 로 바꾼다. 패턴:

```
const tagBase = <기준 리스트>.filter(c => 뷰술어(c) && 성별술어(c) && 검색술어(c))  // selectedTags 제외
const tagGroups = buildTagGroups(tagBase.flatMap(c => c.tags ?? []), tagConfig)
const tCounts = tagCounts(tagBase)
// 렌더용 visible = sort(tagBase.filter(태그술어))  — 기존 visible 계산을 tagBase 재사용 형태로 정리
```

- `<기준 리스트>`: 대부분 `plots`/`chars`/`stories`; **tingle은 `colsByType`(타입탭 적용 후)**, **whif는 `tab`별 `universes`/`characters`**. 각 페이지의 실제 필터 체인을 읽고 "태그 제외 base"를 정확히 분리한다.
- 각 페이지가 이미 가진 `matchesQuery`/`matchesGender`(또는 인라인 gender 비교)/뷰 술어를 재사용.
- `<TagFilterBar ... storageKey="<storagePrefix>_tagcollapse" />` 전달. `explore/all`도 동일(`storageKey="all_tagcollapse"`).

> 주의: tingle은 `typeTab !== 'character'`일 때 성별 필터가 비활성이므로 그 분기를 보존한다. whif는 universe/character 탭에 따라 base가 갈린다.

### 3. 가이드 동기화

- **`context/adding-a-center.md`**: "2. 수동 작업 > E. 탐색/목록 부가 표시"에 항목 추가 — 리스트 페이지는 공용 `TagFilterBar`(접기·`storageKey`)를 쓰고, `tagGroups`/`tCounts`를 **뷰+성별+검색 연동 base**로 계산해야 함(전체 센터/타 센터와 동일). 검증 체크리스트에 "뷰탭 전환 시 해당 상태 카드의 태그만 보이는지" 추가.
- **`CLAUDE.md`**: 규칙 추가 — "센터 공통 검색/필터/태그 동작을 추가·변경하면 `context/adding-a-center.md`도 함께 갱신한다(가이드가 새 센터에 자동 전파되도록)."

## 영향 파일

- `components/ui/TagFilterBar.tsx` (접기 + storageKey)
- 9개 센터 리스트 페이지: `app/(zeta)/zeta/page.tsx`, `(melting)`, `(tikita)`, `(chub)`, `(rofan)`, `(loveydovey)`, `(babechat)`, `(tingle)`, `(whif)/whif/page.tsx`
- `app/(main)/explore/all/page.tsx` (storageKey만; 태그 base는 이미 뷰+성별+검색 연동)
- `context/adding-a-center.md`, `CLAUDE.md`

## 알려진 제약 / 비고

- 페이지네이션은 클라이언트 로드된 카드 기준으로 필터(기존 동작 유지) — 태그 base도 로드된 집합 기준.
- 접기 상태 키는 센터 `storagePrefix` 기반(`zeta_tagcollapse` 등)으로 충돌 없음.

## 범위 외 (YAGNI)

- 태그 그룹 가상화/검색, 태그 즐겨찾기 등은 이번 범위 밖.
