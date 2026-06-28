# 센터 리스트 개선 (정확 카운트·가상 스크롤·뒤로가기 복원) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** loveydovey 센터 리스트를 "경량 인덱스 전체 로드 → 클라이언트 카운트/필터/정렬 → 가상 스크롤 렌더 → 정확한 뒤로가기 복원"으로 재구성하고, 재사용 가능한 공용 훅/컴포넌트로 추출한다.

**Architecture:** 순수 로직(가상 윈도우 계산, 필터/카운트 셀렉터)을 `lib/`로 분리해 vitest로 TDD하고, 그 위에 React 컴포넌트(`VirtualCardGrid`)와 훅(`useCenterList`)을 얹는다. API는 `fields=index` 경량 모드를 추가한다. loveydovey 페이지가 이 인프라를 처음 채택한다(나머지 센터 전파는 후속).

**Tech Stack:** Next.js 14 (App Router, client components), React, TypeScript, Prisma, vitest (node 환경, `lib/**/*.test.ts`만).

## Global Constraints

- 테스트 러너: `npm test` = `vitest run`. vitest 환경은 `node`, include는 `lib/**/*.test.ts`뿐. **React 컴포넌트/훅/Next API 라우트는 vitest로 단위 테스트 불가** → 순수 로직만 TDD, UI/라우트는 `npm run build`(타입체크) + 문서화된 수동 검증.
- 모든 `apps/web` 코드 작업은 `/home/server/StoryFit/apps/web` 하위에서 수행하고, 커밋도 그 서브모듈(`main` 브랜치)에서 한다.
- AI 키 서버 격리 등 CLAUDE.md 규칙 준수. 본 작업은 AI 키와 무관.
- 카운트 기준: 진행/대기/완결 탭 숫자는 **항상 전체 총합**(검색·태그·성별 필터 무관). 태그칩/성별칩 카운트는 필터 반영(기존 동작 유지).
- 가상화 전제: 카드 **고정 높이**(균일 그리드). 측정형 가상화 라이브러리 의존성 추가 금지.
- 경로 alias `@` = `apps/web` 루트(`vitest.config.ts`/`tsconfig.json`).
- 본 플랜 범위는 loveydovey 한 센터 + 공용 인프라까지. 나머지 8개 센터 전파는 후속 플랜.

---

## File Structure

- Create: `lib/virtualWindow.ts` — 순수 가상 스크롤 계산(`computeRowHeight`, `computeVirtualWindow`).
- Create: `lib/virtualWindow.test.ts` — 위 테스트.
- Create: `lib/centerListSelect.ts` — 순수 셀렉터(`selectCenterList`)와 공용 타입(`CenterListItem`, `CenterListFilter`, `CenterListView`).
- Create: `lib/centerListSelect.test.ts` — 위 테스트.
- Modify: `app/api/collections/route.ts` — `fields=index` 경량 전체 로드 모드 추가.
- Create: `components/ui/VirtualCardGrid.tsx` — 측정 기반 2열 윈도잉 그리드(컨테이너 폭 측정 → rowHeight 산출 → `computeVirtualWindow`).
- Create: `lib/useCenterList.ts` — 인덱스 fetch + 모듈 캐시 + sessionStorage 상태/스크롤 복원 + `selectCenterList` 파생.
- Modify: `app/(loveydovey)/loveydovey/page.tsx` — 위 훅/컴포넌트로 전환.
- Modify: `app/globals.css` — `.lovey-card`/`.lovey-card-body`/`.lovey-card-title` 고정 높이.

---

## Task 1: 가상 스크롤 순수 계산 (`lib/virtualWindow.ts`)

**Files:**
- Create: `lib/virtualWindow.ts`
- Test: `lib/virtualWindow.test.ts`

**Interfaces:**
- Consumes: 없음.
- Produces:
  - `interface VirtualWindow { startIndex: number; endIndex: number; topPad: number; bottomPad: number; totalHeight: number }`
  - `function computeRowHeight(p: { containerWidth: number; columns: number; gap: number; padX: number; imageHeightRatio: number; bodyHeight: number }): number` — 한 행 높이(카드 높이 + gap) px.
  - `function computeVirtualWindow(p: { itemCount: number; columns: number; rowHeight: number; scrollTop: number; viewportHeight: number; overscanRows?: number }): VirtualWindow`

- [ ] **Step 1: Write the failing test**

`lib/virtualWindow.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { computeRowHeight, computeVirtualWindow } from './virtualWindow'

describe('computeRowHeight', () => {
  it('컨테이너 폭에서 열 폭을 빼고 이미지비율+본문+gap으로 행 높이를 구한다', () => {
    // width 400, padX16*2=32, gap12, 2열 → colWidth=(400-32-12)/2=178
    // 이미지높이=178*(4/3)=237.33, +본문104=341.33, +gap12=353.33
    const h = computeRowHeight({ containerWidth: 400, columns: 2, gap: 12, padX: 16, imageHeightRatio: 4 / 3, bodyHeight: 104 })
    expect(Math.round(h)).toBe(353)
  })
})

describe('computeVirtualWindow', () => {
  const base = { columns: 2, rowHeight: 100, viewportHeight: 350, overscanRows: 1 }

  it('빈 목록은 0 윈도우와 0 높이', () => {
    const w = computeVirtualWindow({ ...base, itemCount: 0, scrollTop: 0 })
    expect(w).toEqual({ startIndex: 0, endIndex: 0, topPad: 0, bottomPad: 0, totalHeight: 0 })
  })

  it('맨 위에서는 0행부터 보이는 행+오버스캔까지 렌더', () => {
    // 10개=5행, rowHeight100, viewport350 → 보이는행 ceil(350/100)+1=5, +overscan1 → endRow=min(5,0+5+1)=5
    const w = computeVirtualWindow({ ...base, itemCount: 10, scrollTop: 0 })
    expect(w.startIndex).toBe(0)
    expect(w.endIndex).toBe(10)
    expect(w.topPad).toBe(0)
    expect(w.bottomPad).toBe(0)
    expect(w.totalHeight).toBe(500)
  })

  it('중간 스크롤은 위쪽 행을 잘라내고 topPad를 채운다', () => {
    // 100개=50행, scrollTop 1000 → firstVisibleRow=10, startRow=10-1=9, topPad=900
    const w = computeVirtualWindow({ ...base, itemCount: 100, scrollTop: 1000 })
    expect(w.startIndex).toBe(18) // 9행 * 2열
    expect(w.topPad).toBe(900)    // 9행 * 100
    expect(w.totalHeight).toBe(5000)
    expect(w.bottomPad).toBe(5000 - w.endIndex / 2 * 100)
  })

  it('마지막 행을 넘어 스크롤해도 인덱스가 itemCount를 넘지 않는다', () => {
    const w = computeVirtualWindow({ ...base, itemCount: 7, scrollTop: 99999 })
    expect(w.endIndex).toBe(7)
    expect(w.bottomPad).toBe(0)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/virtualWindow.test.ts`
Expected: FAIL — `computeRowHeight`/`computeVirtualWindow` is not defined.

- [ ] **Step 3: Write minimal implementation**

`lib/virtualWindow.ts`:

```ts
// 센터 리스트 가상 스크롤의 순수 계산. 측정값(폭/스크롤/뷰포트)을 받아 렌더 윈도우를 돌려준다.

export interface VirtualWindow {
  startIndex: number  // 렌더 시작 항목 인덱스(포함)
  endIndex: number    // 렌더 끝 항목 인덱스(제외)
  topPad: number      // 윈도우 위 스페이서 높이 px
  bottomPad: number   // 윈도우 아래 스페이서 높이 px
  totalHeight: number // 전체 행 높이 합 px
}

// 반응형 이미지(aspect-ratio) 카드의 한 행 높이. 열 폭에서 카드 폭을 구하고
// 이미지 높이(폭*비율) + 고정 본문 + gap 을 더한다.
export function computeRowHeight(p: {
  containerWidth: number; columns: number; gap: number; padX: number
  imageHeightRatio: number; bodyHeight: number
}): number {
  const inner = p.containerWidth - 2 * p.padX - (p.columns - 1) * p.gap
  const colWidth = Math.max(0, inner / p.columns)
  return colWidth * p.imageHeightRatio + p.bodyHeight + p.gap
}

export function computeVirtualWindow(p: {
  itemCount: number; columns: number; rowHeight: number
  scrollTop: number; viewportHeight: number; overscanRows?: number
}): VirtualWindow {
  const overscan = p.overscanRows ?? 2
  const totalRows = Math.ceil(p.itemCount / p.columns)
  const totalHeight = totalRows * p.rowHeight
  if (p.itemCount === 0 || p.rowHeight <= 0) {
    return { startIndex: 0, endIndex: 0, topPad: 0, bottomPad: 0, totalHeight: 0 }
  }
  const firstVisibleRow = Math.floor(p.scrollTop / p.rowHeight)
  const visibleRowCount = Math.ceil(p.viewportHeight / p.rowHeight) + 1
  const startRow = Math.max(0, firstVisibleRow - overscan)
  const endRow = Math.min(totalRows, firstVisibleRow + visibleRowCount + overscan)
  const startIndex = startRow * p.columns
  const endIndex = Math.min(p.itemCount, endRow * p.columns)
  return {
    startIndex,
    endIndex,
    topPad: startRow * p.rowHeight,
    bottomPad: Math.max(0, (totalRows - endRow) * p.rowHeight),
    totalHeight,
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/virtualWindow.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add lib/virtualWindow.ts lib/virtualWindow.test.ts && git commit -m "feat: 센터 리스트 가상 스크롤 순수 계산(virtualWindow)"
```

---

## Task 2: 카운트/필터 셀렉터 (`lib/centerListSelect.ts`)

**Files:**
- Create: `lib/centerListSelect.ts`
- Test: `lib/centerListSelect.test.ts`

**Interfaces:**
- Consumes: `sortByOption`/`SortOption`(`lib/listSort`), `viewCounts`/`tagCounts`/`ViewCounts`(`lib/centerCounts`), `buildTagGroups`/`CenterTagConfig`(`lib/tagGroups`), `availableGenderBuckets`/`cardGenderBucket`(`lib/cardGender`).
- Produces:
  - `interface CenterListItem { id: string; title: string; coverImageUrl?: string; description?: string; tags: string[]; createdAt?: string; lastActivityAt?: string; completed?: boolean; started?: boolean; characters: { id: string; name: string; avatarUrl: string | null; gender?: string | null }[] }`
  - `interface CenterListFilter { view: 'active' | 'waiting' | 'completed' | 'favorites'; sort: SortOption; query: string; selectedTags: string[]; genderFilter: string; randomSeed: number }`
  - `interface CenterListView { counts: ViewCounts; tagGroups: ReturnType<typeof buildTagGroups>; tCounts: Record<string, number>; genderBuckets: ReturnType<typeof availableGenderBuckets>; visibleChars: CenterListItem[] }`
  - `function selectCenterList(items: CenterListItem[], filter: CenterListFilter, tagConfig: CenterTagConfig | null, isFav: (id: string) => boolean): CenterListView`

- [ ] **Step 1: Write the failing test**

`lib/centerListSelect.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { selectCenterList, type CenterListItem, type CenterListFilter } from './centerListSelect'

const mk = (over: Partial<CenterListItem>): CenterListItem => ({
  id: 'x', title: 't', tags: [], characters: [], ...over,
})

const items: CenterListItem[] = [
  mk({ id: 'a', title: '가', tags: ['로맨스'], started: true, completed: false, createdAt: '2026-01-01', characters: [{ id: 'c1', name: 'A', avatarUrl: null, gender: 'female' }] }),
  mk({ id: 'b', title: '나', tags: ['로맨스', '판타지'], started: false, completed: false, createdAt: '2026-02-01', characters: [{ id: 'c2', name: 'B', avatarUrl: null, gender: 'male' }] }),
  mk({ id: 'c', title: '다', tags: ['판타지'], started: true, completed: true, createdAt: '2026-03-01', characters: [{ id: 'c3', name: 'C', avatarUrl: null, gender: 'female' }] }),
]
const baseFilter: CenterListFilter = { view: 'active', sort: 'latest', query: '', selectedTags: [], genderFilter: 'all', randomSeed: 0 }
const noFav = () => false

describe('selectCenterList', () => {
  it('counts는 필터와 무관하게 전체 총합', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'active', query: '가', selectedTags: ['로맨스'] }, null, noFav)
    expect(v.counts).toEqual({ active: 1, waiting: 1, completed: 1 })
  })

  it('view=active는 시작했고 미완결인 항목만', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'active' }, null, noFav)
    expect(v.visibleChars.map(i => i.id)).toEqual(['a'])
  })

  it('view=waiting은 미시작 항목만', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'waiting' }, null, noFav)
    expect(v.visibleChars.map(i => i.id)).toEqual(['b'])
  })

  it('view=completed는 완결 항목만', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'completed' }, null, noFav)
    expect(v.visibleChars.map(i => i.id)).toEqual(['c'])
  })

  it('view=favorites는 isFav가 true인 항목만', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'favorites' }, null, id => id === 'b')
    expect(v.visibleChars.map(i => i.id)).toEqual(['b'])
  })

  it('태그 필터는 view 결과를 추가로 좁힌다', () => {
    const v = selectCenterList(items, { ...baseFilter, view: 'completed', selectedTags: ['로맨스'] }, null, noFav)
    expect(v.visibleChars).toEqual([])
  })

  it('정렬 latest는 createdAt 내림차순', () => {
    // completed 탭이 아니라 전체가 보이도록 favorites로 모두 true
    const v = selectCenterList(items, { ...baseFilter, view: 'favorites', sort: 'latest' }, null, () => true)
    expect(v.visibleChars.map(i => i.id)).toEqual(['c', 'b', 'a'])
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/centerListSelect.test.ts`
Expected: FAIL — `selectCenterList` is not defined.

- [ ] **Step 3: Write minimal implementation**

`lib/centerListSelect.ts`:

```ts
// 센터 리스트의 카운트/필터/정렬을 한 번에 계산하는 순수 셀렉터.
// counts는 항상 전체 총합(필터 무관), visibleChars는 view+검색+성별+태그+정렬 적용 결과.
import { sortByOption, type SortOption } from './listSort'
import { viewCounts, tagCounts, type ViewCounts } from './centerCounts'
import { buildTagGroups, type CenterTagConfig } from './tagGroups'
import { availableGenderBuckets, cardGenderBucket } from './cardGender'

export interface CenterListItem {
  id: string
  title: string
  coverImageUrl?: string
  description?: string
  tags: string[]
  createdAt?: string
  lastActivityAt?: string
  completed?: boolean
  started?: boolean
  characters: { id: string; name: string; avatarUrl: string | null; gender?: string | null }[]
}

export interface CenterListFilter {
  view: 'active' | 'waiting' | 'completed' | 'favorites'
  sort: SortOption
  query: string
  selectedTags: string[]
  genderFilter: string // 'all' | GenderBucket
  randomSeed: number
}

export interface CenterListView {
  counts: ViewCounts
  tagGroups: ReturnType<typeof buildTagGroups>
  tCounts: Record<string, number>
  genderBuckets: ReturnType<typeof availableGenderBuckets>
  visibleChars: CenterListItem[]
}

export function selectCenterList(
  items: CenterListItem[],
  filter: CenterListFilter,
  tagConfig: CenterTagConfig | null,
  isFav: (id: string) => boolean,
): CenterListView {
  const { view, sort, query, selectedTags, genderFilter, randomSeed } = filter
  const q = query.trim().toLowerCase()

  const viewMatch = (c: CenterListItem) =>
    view === 'favorites' ? isFav(c.id)
    : view === 'completed' ? !!c.completed
    : view === 'waiting' ? !c.started
    : !c.completed && !!c.started

  const matchesQuery = (c: CenterListItem) =>
    !q || c.title.toLowerCase().includes(q) || (c.tags ?? []).some(t => t.toLowerCase().includes(q))
  const matchesTag = (c: CenterListItem) =>
    selectedTags.length === 0 || selectedTags.every(t => (c.tags ?? []).includes(t))
  const matchesGender = (c: CenterListItem) =>
    genderFilter === 'all' || cardGenderBucket(c.characters) === genderFilter

  // counts: 전체 총합(필터 무관)
  const counts = viewCounts(items)
  const genderBuckets = availableGenderBuckets(items)

  // 태그 목록/카운트 base: view+성별+검색 적용(태그 제외)
  const tagBase = items.filter(c => viewMatch(c) && matchesGender(c) && matchesQuery(c))
  const tagGroups = buildTagGroups(tagBase.flatMap(c => c.tags ?? []), tagConfig)
  const tCounts = tagCounts(tagBase)

  const visibleChars = sortByOption(
    tagBase.filter(matchesTag),
    sort,
    c => c.title,
    c => c.createdAt ?? '',
    c => c.lastActivityAt ?? c.createdAt ?? '',
    randomSeed,
  )

  return { counts, tagGroups, tCounts, genderBuckets, visibleChars }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/centerListSelect.test.ts`
Expected: PASS (7 tests).

- [ ] **Step 5: Run full suite (no regressions)**

Run: `cd /home/server/StoryFit/apps/web && npm test`
Expected: PASS — 기존 테스트(listSort/completion/josa 등) 포함 전체 통과.

- [ ] **Step 6: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add lib/centerListSelect.ts lib/centerListSelect.test.ts && git commit -m "feat: 센터 리스트 카운트/필터 순수 셀렉터(centerListSelect)"
```

---

## Task 3: API `fields=index` 경량 전체 로드 모드

**Files:**
- Modify: `app/api/collections/route.ts`

**Interfaces:**
- Consumes: 없음(기존 라우트 수정).
- Produces: `GET /api/collections?<center>&fields=index` → `CenterListItem[]`(전체, 경량). 항목별 `completed`/`started`/`lastActivityAt`는 서버 집계값. `limit`/`offset` 무시. lorebook/openingMessages/`*Meta` 제외.

- [ ] **Step 1: `isIndex` 플래그 추가**

`app/api/collections/route.ts`에서 `fields=basic` 처리 블록 **직후**, `const limit = ...` 줄 **앞**에 추가:

```ts
  const isIndex = searchParams.get('fields') === 'index'
```

- [ ] **Step 2: findMany를 index 모드에 맞게 분기**

기존 블록:

```ts
  const collections = await prisma.characterCollection.findMany({
    where: whereClause,
    orderBy: { createdAt: 'desc' },
    ...(limit ? { take: limit, skip: offset } : {}),
    select: {
      id: true,
      title: true,
      sourceUrl: true,
      createdAt: true,
      coverImageUrl: true,
      description: true,
      tags: true,
      zetaMeta: true,
      meltingMeta: true,
      tikitaMeta: true,
      characters: { select: { id: true, name: true, avatarUrl: true, gender: true, openingMessage: true, openingMessages: true } },
    },
  })
```

를 다음으로 교체:

```ts
  const collections = await prisma.characterCollection.findMany({
    where: whereClause,
    orderBy: { createdAt: 'desc' },
    ...(isIndex ? {} : (limit ? { take: limit, skip: offset } : {})),
    select: {
      id: true,
      title: true,
      sourceUrl: true,
      createdAt: true,
      coverImageUrl: true,
      description: true,
      tags: true,
      ...(isIndex ? {} : { zetaMeta: true, meltingMeta: true, tikitaMeta: true }),
      characters: {
        select: isIndex
          ? { id: true, name: true, avatarUrl: true, gender: true }
          : { id: true, name: true, avatarUrl: true, gender: true, openingMessage: true, openingMessages: true },
      },
    },
  })
```

- [ ] **Step 3: lorebook 쿼리를 index에서 생략**

기존:

```ts
  const lorebooks = collectionIds.length > 0
    ? await prisma.lorebook.findMany({
```

를:

```ts
  const lorebooks = (!isIndex && collectionIds.length > 0)
    ? await prisma.lorebook.findMany({
```

- [ ] **Step 4: 결과 매핑을 index 모드에 맞게 분기**

기존 매핑 블록:

```ts
  const result = collections.map(c => {
    const convMap = convsByCollection.get(c.id)
    const counts = aggregateCounts(convMap ? Array.from(convMap.values()) : [])
    return {
      ...c,
      lorebookTitles: lorebookTitlesByCollection.get(c.id) ?? [],
      completed: isCompleted(counts),
      started: counts.activeCount + counts.archivedCount > 0,
      lastActivityAt: lastActivityByCollection.get(c.id) ?? c.createdAt,
      characters: c.characters.map(ch => ({ ...ch, hasArchived: archivedCharIds.has(ch.id) })),
    }
  })
  return NextResponse.json(result)
```

를:

```ts
  const result = collections.map(c => {
    const convMap = convsByCollection.get(c.id)
    const counts = aggregateCounts(convMap ? Array.from(convMap.values()) : [])
    const base = {
      ...c,
      completed: isCompleted(counts),
      started: counts.activeCount + counts.archivedCount > 0,
      lastActivityAt: lastActivityByCollection.get(c.id) ?? c.createdAt,
    }
    if (isIndex) return base
    return {
      ...base,
      lorebookTitles: lorebookTitlesByCollection.get(c.id) ?? [],
      characters: c.characters.map(ch => ({ ...ch, hasArchived: archivedCharIds.has(ch.id) })),
    }
  })
  return NextResponse.json(result)
```

- [ ] **Step 5: 타입체크/빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: 빌드 성공(타입 에러 없음). (lint 경고는 무방, 에러 없으면 통과)

- [ ] **Step 6: 수동 검증**

개발 서버 기동 후(로그인 상태) 브라우저 콘솔/네트워크에서 확인:

```js
// 브라우저 devtools 콘솔 (로그인 토큰이 쿠키/헤더에 있는 환경)
fetch('/api/collections?isLoveydovey=true&fields=index', { headers: { Authorization: `Bearer ${localStorage.getItem('accessToken')}` } })
  .then(r => r.json()).then(d => console.log('count', d.length, d[0]))
```

Expected: 배열 길이가 페이지네이션 없이 전체 개수와 일치하고, 각 항목에 `completed`/`started`/`lastActivityAt`/`tags`/`characters[{id,name,avatarUrl,gender}]`가 있으며 `zetaMeta`/`openingMessages`/`lorebookTitles`는 **없다**.

> 참고: 토큰 취득 방식이 다르면 `lib/api.ts`의 인증 방식에 맞춰 호출. 핵심은 길이가 전체수와 같고 경량 필드만 오는지.

- [ ] **Step 7: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add app/api/collections/route.ts && git commit -m "feat: /api/collections fields=index 경량 전체 로드 모드"
```

---

## Task 4: 가상 카드 그리드 (`components/ui/VirtualCardGrid.tsx`)

**Files:**
- Create: `components/ui/VirtualCardGrid.tsx`

**Interfaces:**
- Consumes: `computeRowHeight`, `computeVirtualWindow`(Task 1).
- Produces: 기본 export `VirtualCardGrid<T>(props)`:
  ```ts
  {
    items: T[]
    renderItem: (item: T) => React.ReactNode
    scrollRef: React.RefObject<HTMLElement | null>  // 스크롤 컨테이너
    imageHeightRatio: number  // 카드 이미지 높이/폭 (aspect-ratio:3/4 → 4/3)
    bodyHeight: number        // 카드 본문 고정 높이 px
    columns?: number          // 기본 2
    gap?: number              // 기본 12
    padX?: number             // 그리드 좌우 패딩, 기본 16
    overscanRows?: number     // 기본 2
  }
  ```

- [ ] **Step 1: 컴포넌트 작성**

`components/ui/VirtualCardGrid.tsx`:

```tsx
'use client'
import { useLayoutEffect, useState, type ReactNode, type RefObject } from 'react'
import { computeRowHeight, computeVirtualWindow } from '@/lib/virtualWindow'

export default function VirtualCardGrid<T>({
  items, renderItem, scrollRef,
  imageHeightRatio, bodyHeight,
  columns = 2, gap = 12, padX = 16, overscanRows = 2,
}: {
  items: T[]
  renderItem: (item: T) => ReactNode
  scrollRef: RefObject<HTMLElement | null>
  imageHeightRatio: number
  bodyHeight: number
  columns?: number
  gap?: number
  padX?: number
  overscanRows?: number
}) {
  const [metrics, setMetrics] = useState({ scrollTop: 0, viewportHeight: 0, containerWidth: 0 })

  // 레이아웃 단계에서 측정 → 첫 페인트부터 총높이/윈도우가 정확.
  useLayoutEffect(() => {
    const el = scrollRef.current
    if (!el) return
    const read = () => setMetrics({ scrollTop: el.scrollTop, viewportHeight: el.clientHeight, containerWidth: el.clientWidth })
    read()
    el.addEventListener('scroll', read, { passive: true })
    window.addEventListener('resize', read)
    return () => { el.removeEventListener('scroll', read); window.removeEventListener('resize', read) }
  }, [scrollRef])

  const rowHeight = metrics.containerWidth > 0
    ? computeRowHeight({ containerWidth: metrics.containerWidth, columns, gap, padX, imageHeightRatio, bodyHeight })
    : 0
  const win = computeVirtualWindow({
    itemCount: items.length, columns, rowHeight,
    scrollTop: metrics.scrollTop, viewportHeight: metrics.viewportHeight, overscanRows,
  })
  const slice = items.slice(win.startIndex, win.endIndex)

  return (
    <div style={{ paddingTop: win.topPad, paddingBottom: win.bottomPad }}>
      <div style={{ display: 'grid', gridTemplateColumns: `repeat(${columns}, 1fr)`, gap, padding: `0 ${padX}px` }}>
        {slice.map(renderItem)}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: 타입체크/빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: 빌드 성공(타입 에러 없음).

- [ ] **Step 3: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add components/ui/VirtualCardGrid.tsx && git commit -m "feat: 측정 기반 가상 카드 그리드(VirtualCardGrid)"
```

---

## Task 5: 센터 리스트 훅 (`lib/useCenterList.ts`)

**Files:**
- Create: `lib/useCenterList.ts`

**Interfaces:**
- Consumes: `api`(`lib/api`), `selectCenterList`/`CenterListItem`/`CenterListFilter`(Task 2), `useFavorites`(`lib/useFavorites`), `CenterTagConfig`(`lib/tagGroups`), `SortOption`(`lib/listSort`).
- Produces: `function useCenterList(opts: { indexQuery: string; storagePrefix: string }) => {`
  - `items: CenterListItem[]`, `loading: boolean`
  - `view`, `setView(v)`, `sort`, `setSort(v)`, `query`, `setQuery(v)`,
  - `selectedTags: string[]`, `toggleTag(t)`, `clearTags()`, `genderFilter: string`, `setGenderFilter(v)`,
  - `searchOpen: boolean`, `toggleSearch()`, `randomSeed: number`,
  - `counts`, `tagGroups`, `tCounts`, `genderBuckets`, `visibleChars`(=`selectCenterList` 결과),
  - `isFav(type,id)`, `toggleFav(type,id)`(useFavorites 패스스루),
  - `scrollRef: React.RefObject<HTMLDivElement | null>`(스크롤 컨테이너에 부착),
  - `refresh(): Promise<void>`(인덱스 강제 재로드; import/생성/삭제 후 호출)
  - `}`
  - `tagConfig`는 내부에서 fetch하여 selectCenterList에 주입(외부 노출 불필요).

- [ ] **Step 1: 훅 작성**

`lib/useCenterList.ts`:

```ts
'use client'
import { useCallback, useEffect, useRef, useState } from 'react'
import { api } from './api'
import { useFavorites } from './useFavorites'
import { selectCenterList, type CenterListItem, type CenterListFilter } from './centerListSelect'
import type { CenterTagConfig } from './tagGroups'
import type { SortOption } from './listSort'

type View = CenterListFilter['view']

// 센터별 인덱스 전체 배열을 모듈 스코프에 캐시 → 클라 네비게이션(뒤로가기) 간 즉시 복원.
const indexCache = new Map<string, CenterListItem[]>()

export function useCenterList(opts: { indexQuery: string; storagePrefix: string }) {
  const { indexQuery, storagePrefix } = opts
  const cacheKey = indexQuery

  const { isFav, toggleFav } = useFavorites()
  const [items, setItems] = useState<CenterListItem[]>(() => indexCache.get(cacheKey) ?? [])
  const [loading, setLoading] = useState(() => !indexCache.has(cacheKey))
  const [tagConfig, setTagConfig] = useState<CenterTagConfig | null>(null)

  // 필터 상태 (초기값은 sessionStorage 복원)
  const [view, setViewState] = useState<View>('active')
  const [sort, setSortState] = useState<SortOption>('latest')
  const [query, setQuery] = useState('')
  const [selectedTags, setSelectedTags] = useState<string[]>([])
  const [genderFilter, setGenderFilter] = useState('all')
  const [searchOpen, setSearchOpen] = useState(false)
  const [randomSeed, setRandomSeed] = useState(() => Math.floor(Math.random() * 1e9))

  const scrollRef = useRef<HTMLDivElement>(null)
  const restored = useRef(false)

  // 초기 상태 복원 + 인덱스 fetch (캐시 있으면 즉시 사용)
  useEffect(() => {
    setView((sessionStorage.getItem(`${storagePrefix}_view`) as View) || 'active')
    setSortState((localStorage.getItem(`${storagePrefix}_sort`) as SortOption) || 'latest')
    try {
      const raw = sessionStorage.getItem(`${storagePrefix}_filter`)
      if (raw) {
        const f = JSON.parse(raw)
        if (typeof f.query === 'string') setQuery(f.query)
        if (Array.isArray(f.selectedTags)) setSelectedTags(f.selectedTags)
        if (typeof f.genderFilter === 'string') setGenderFilter(f.genderFilter)
        if (typeof f.searchOpen === 'boolean') setSearchOpen(f.searchOpen)
        if (typeof f.randomSeed === 'number') setRandomSeed(f.randomSeed)
      }
    } catch {}
    api.get('/api/center-tags').then(setTagConfig).catch(() => {})
    if (!indexCache.has(cacheKey)) void load()
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [])

  const load = useCallback(async () => {
    setLoading(true)
    try {
      const data: CenterListItem[] = await api.get(`/api/collections?${indexQuery}&fields=index`)
      indexCache.set(cacheKey, data)
      setItems(data)
    } finally { setLoading(false) }
  }, [indexQuery, cacheKey])

  const refresh = useCallback(async () => { await load() }, [load])

  // 필터 상태 영속화
  useEffect(() => { sessionStorage.setItem(`${storagePrefix}_view`, view) }, [view, storagePrefix])
  useEffect(() => {
    sessionStorage.setItem(`${storagePrefix}_filter`, JSON.stringify({ query, selectedTags, genderFilter, searchOpen, randomSeed }))
  }, [query, selectedTags, genderFilter, searchOpen, randomSeed, storagePrefix])

  // 스크롤 위치 저장 + 복원 (전체 항목이 메모리에 있으므로 가상화 총높이가 정확 → 복원 신뢰성 높음)
  useEffect(() => {
    const el = scrollRef.current
    if (!el) return
    const onScroll = () => sessionStorage.setItem(`${storagePrefix}_scroll`, String(el.scrollTop))
    el.addEventListener('scroll', onScroll, { passive: true })
    return () => el.removeEventListener('scroll', onScroll)
  }, [storagePrefix])

  useEffect(() => {
    if (loading || restored.current) return
    const el = scrollRef.current
    if (!el) return
    const saved = sessionStorage.getItem(`${storagePrefix}_scroll`)
    if (saved) requestAnimationFrame(() => { el.scrollTop = parseInt(saved, 10) })
    restored.current = true
  }, [loading, storagePrefix])

  // 핸들러
  const setView = useCallback((v: View) => setViewState(v), [])
  const setSort = useCallback((v: SortOption) => {
    setSortState(v); localStorage.setItem(`${storagePrefix}_sort`, v)
    if (v === 'random') setRandomSeed(Math.floor(Math.random() * 1e9))
  }, [storagePrefix])
  const toggleTag = useCallback((t: string) => setSelectedTags(prev => prev.includes(t) ? prev.filter(x => x !== t) : [...prev, t]), [])
  const clearTags = useCallback(() => setSelectedTags([]), [])
  const toggleSearch = useCallback(() => setSearchOpen(o => {
    if (o) { setQuery(''); setSelectedTags([]); setGenderFilter('all') }
    return !o
  }), [])

  const { counts, tagGroups, tCounts, genderBuckets, visibleChars } = selectCenterList(
    items,
    { view, sort, query, selectedTags, genderFilter, randomSeed },
    tagConfig,
    (id: string) => isFav('collection', id),
  )

  return {
    items, loading,
    view, setView, sort, setSort, query, setQuery,
    selectedTags, toggleTag, clearTags, genderFilter, setGenderFilter,
    searchOpen, toggleSearch, randomSeed,
    counts, tagGroups, tCounts, genderBuckets, visibleChars,
    isFav, toggleFav, scrollRef, refresh,
  }
}
```

- [ ] **Step 2: 타입체크/빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: 빌드 성공(타입 에러 없음).

- [ ] **Step 3: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add lib/useCenterList.ts && git commit -m "feat: 센터 리스트 공용 훅(useCenterList) — 인덱스 캐시·상태 영속·스크롤 복원"
```

---

## Task 6: loveydovey 페이지 전환 + 고정 높이 카드 CSS

**Files:**
- Modify: `app/(loveydovey)/loveydovey/page.tsx`
- Modify: `app/globals.css`

**Interfaces:**
- Consumes: `useCenterList`(Task 5), `VirtualCardGrid`(Task 4), 기존 `TagFilterBar`, `replaceDisplayPlaceholders`.
- Produces: 없음(말단).

- [ ] **Step 1: 카드 고정 높이 CSS 적용**

`app/globals.css`의 loveydovey 카드 규칙(현재 `~1386`행 부근)을 수정. 기존:

```css
.lovey-card-body{ padding:10px; display:flex; flex-direction:column; gap:6px; }
.lovey-card-title{ font-size:13px; font-weight:800; color:var(--l-ink); overflow:hidden;
```

를 다음으로 교체(본문 고정 높이 + 제목 1줄 클램프). 기존 `.lovey-card-title`의 나머지 선언(말줄임 등)은 유지하되 `-webkit-line-clamp`를 1로:

```css
.lovey-card-body{ padding:10px; display:flex; flex-direction:column; gap:6px;
  height:104px; box-sizing:border-box; overflow:hidden; }
.lovey-card-title{ font-size:13px; font-weight:800; color:var(--l-ink); overflow:hidden;
  display:-webkit-box; -webkit-line-clamp:1; -webkit-box-orient:vertical; }
```

> 참고: 기존 `.lovey-card-title`에 이미 `display:-webkit-box; -webkit-line-clamp:2;` 등이 있으면 그 값을 1로 바꾸기만 한다(중복 선언 추가 금지). 정확한 px(104)는 Step 5에서 실측 보정.

카드 본문 내 설명 줄 클램프는 페이지의 인라인 스타일(`WebkitLineClamp: 2`)로 유지된다.

- [ ] **Step 2: 페이지를 훅/가상그리드로 전환**

`app/(loveydovey)/loveydovey/page.tsx` 전체를 다음으로 교체:

```tsx
'use client'
import { useEffect, useState } from 'react'
import { useRouter } from 'next/navigation'
import { api } from '@/lib/api'
import TagFilterBar from '@/components/ui/TagFilterBar'
import VirtualCardGrid from '@/components/ui/VirtualCardGrid'
import { useCenterList } from '@/lib/useCenterList'
import { replaceDisplayPlaceholders } from '@/lib/josa'
import type { CenterListItem } from '@/lib/centerListSelect'

export default function LoveydoveyListPage() {
  const router = useRouter()
  const {
    items, loading,
    view, setView, sort, setSort, query, setQuery,
    selectedTags, toggleTag, clearTags, genderFilter, setGenderFilter,
    searchOpen, toggleSearch,
    counts, tagGroups, tCounts, genderBuckets, visibleChars,
    isFav, toggleFav, scrollRef, refresh,
  } = useCenterList({ indexQuery: 'isLoveydovey=true', storagePrefix: 'lovey' })

  const [menuOpen, setMenuOpen] = useState(false)
  const [editMode, setEditMode] = useState(false)
  const [importUrl, setImportUrl] = useState('')
  const [importing, setImporting] = useState(false)
  const [msg, setMsg] = useState('')

  useEffect(() => { setEditMode(localStorage.getItem('lovey_edit') === '1') }, [])

  const handleImport = async () => {
    const urls = importUrl.split(String.fromCharCode(10)).map(u => u.trim()).filter(Boolean)
    if (urls.length === 0 || importing) return
    setImporting(true)
    let ok = 0
    const failed: string[] = []
    for (let i = 0; i < urls.length; i++) {
      setMsg(`가져오는 중... (${i + 1}/${urls.length})`)
      try { await api.post('/api/characters/import', { url: urls[i] }); ok++ }
      catch { failed.push(urls[i]) }
    }
    setImportUrl(failed.join(String.fromCharCode(10)))
    setMsg(failed.length ? `✓ ${ok}개 완료 · ⚠ ${failed.length}개 실패 — 다시 가져오기로 재시도` : `✓ ${ok}개 가져왔습니다`)
    if (failed.length === 0) setMenuOpen(false)
    await refresh()
    setImporting(false)
  }

  const toggleEditMode = () => {
    const next = !editMode; setEditMode(next)
    localStorage.setItem('lovey_edit', next ? '1' : '0'); setMenuOpen(false)
  }

  const createCharacter = async () => {
    const title = prompt('새 캐릭터 이름'); if (!title?.trim()) return
    await api.post('/api/collections', { title: title.trim(), sourceUrl: `https://loveydovey.ai/local/${Date.now()}` })
    setMenuOpen(false); await refresh()
  }

  const deleteChar = async (id: string) => {
    if (!confirm('이 캐릭터를 삭제할까요?')) return
    await api.delete(`/api/collections/${id}`); await refresh()
  }

  const renderCard = (c: CenterListItem) => {
    const thumb = c.coverImageUrl || c.characters[0]?.avatarUrl || ''
    return (
      <div key={c.id} className="lovey-card" style={{ position: 'relative' }}
        onClick={() => !editMode && router.push(`/loveydovey/characters/${c.id}`)}>
        {c.completed && <div style={{ position: 'absolute', top: 6, left: 6, zIndex: 2, fontSize: 9, fontWeight: 700, background: 'var(--l-accent)', color: '#fff', padding: '1px 5px', borderRadius: 3 }}>완결</div>}
        {thumb ? <img className="lovey-card-img" loading="lazy" decoding="async" src={thumb} alt="" /> : <div className="lovey-card-img" />}
        <div className="lovey-card-body">
          <div className="lovey-card-title">{c.title}</div>
          {c.description?.trim() && (
            <div style={{ fontSize: 11, color: 'var(--l-ink-soft)', lineHeight: 1.4, overflow: 'hidden', textOverflow: 'ellipsis', display: '-webkit-box', WebkitLineClamp: 2, WebkitBoxOrient: 'vertical' }}>
              {replaceDisplayPlaceholders(c.description, '나', c.characters?.[0]?.name ?? '')}
            </div>
          )}
          {c.tags?.length > 0 && (
            <div className="lovey-card-tags">
              {c.tags.slice(0, 3).map(t => <span key={t} className="lovey-chip">#{t}</span>)}
            </div>
          )}
        </div>
        {editMode ? (
          <button onClick={e => { e.stopPropagation(); deleteChar(c.id) }}
            style={{ position: 'absolute', top: 6, right: 6, background: 'rgba(0,0,0,0.7)',
              border: 'none', color: '#ff6b8a', borderRadius: 999, width: 24, height: 24,
              cursor: 'pointer', fontSize: 14, display: 'flex', alignItems: 'center', justifyContent: 'center' }}>✕</button>
        ) : (
          <button onClick={e => { e.stopPropagation(); toggleFav('collection', c.id) }}
            aria-label="즐겨찾기"
            style={{ position: 'absolute', top: 6, right: 6, background: 'rgba(0,0,0,0.55)',
              border: 'none', color: isFav('collection', c.id) ? '#ffd24a' : '#fff', borderRadius: 999, width: 24, height: 24,
              cursor: 'pointer', fontSize: 13, display: 'flex', alignItems: 'center', justifyContent: 'center' }}>{isFav('collection', c.id) ? '★' : '☆'}</button>
        )}
      </div>
    )
  }

  return (
    <>
      <div className="lovey-header" style={{ position: 'relative' }}>
        <div style={{ display: 'flex', alignItems: 'center', gap: 4 }}>
          <button className="lovey-iconbtn" aria-label="홈으로" onClick={() => router.push('/')}>🏠</button>
          <div className="lovey-logo">loveydovey</div>
        </div>
        <button className="lovey-iconbtn" onClick={() => setMenuOpen(o => !o)}>⋮</button>
        {menuOpen && (
          <div className="lovey-menu">
            <div style={{ padding: '10px 10px 4px', display: 'flex', flexDirection: 'column', gap: 4 }}>
              <textarea className="field" placeholder="URL을 한 줄에 하나씩 붙여넣기 (여러 개 가능)" value={importUrl} onChange={e => setImportUrl(e.target.value)} rows={3} style={{ fontSize: 12, resize: 'vertical' }} />
              <button className="lovey-menu-item"
                style={{ background: 'var(--l-accent)', borderRadius: 8, color: '#fff', textAlign: 'center' }}
                disabled={importing} onClick={handleImport}>{importing ? '가져오는 중...' : '📥 가져오기 (메타데이터)'}</button>
            </div>
            <button className="lovey-menu-item" onClick={createCharacter}>+ 새 캐릭터 만들기</button>
            <button className="lovey-menu-item" onClick={toggleEditMode}>
              {editMode ? '✓ 편집 모드 끄기' : '✏ 편집 모드 켜기'}
            </button>
          </div>
        )}
      </div>

      {msg && <div style={{ padding: '6px 16px', fontSize: 12, color: msg.startsWith('✓') ? '#4ade80' : '#ff6b8a' }}>{msg}</div>}

      <div style={{ display: 'flex', gap: 6, padding: '8px 16px', alignItems: 'center', justifyContent: 'space-between' }}>
        <div style={{ display: 'flex', gap: 6 }}>
          <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: view === 'active' ? 'var(--l-accent)' : 'var(--l-surface-2)', color: view === 'active' ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setView('active')}>진행 중 <span style={{ opacity: 0.55 }}>{counts.active}</span></button>
          <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: view === 'waiting' ? 'var(--l-accent)' : 'var(--l-surface-2)', color: view === 'waiting' ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setView('waiting')}>대기 <span style={{ opacity: 0.55 }}>{counts.waiting}</span></button>
          <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: view === 'completed' ? 'var(--l-accent)' : 'var(--l-surface-2)', color: view === 'completed' ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setView('completed')}>완결 <span style={{ opacity: 0.55 }}>{counts.completed}</span></button>
          <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: view === 'favorites' ? 'var(--l-accent)' : 'var(--l-surface-2)', color: view === 'favorites' ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setView('favorites')}>★ 즐겨찾기</button>
        </div>
        <div style={{ display: 'flex', gap: 6, alignItems: 'center' }}>
          <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: searchOpen ? 'var(--l-accent)' : 'var(--l-surface-2)', color: searchOpen ? '#fff' : 'var(--l-ink-soft)' }} onClick={toggleSearch}>🔍 검색</button>
          <select
            className="field"
            style={{ fontSize: 11, padding: '2px 6px', width: 'auto' }}
            value={sort}
            onChange={e => setSort(e.target.value as typeof sort)}
          >
            <option value="latest">최신순</option>
            <option value="oldest">오래된순</option>
            <option value="alpha">가나다순</option>
            <option value="active">최근 대화순</option>
            <option value="random">🔀 랜덤</option>
          </select>
        </div>
      </div>

      {searchOpen && (
        <>
          <div style={{ padding: '0 16px 8px' }}>
            <input
              className="field"
              style={{ fontSize: 12, width: '100%' }}
              placeholder="이름으로 검색"
              value={query}
              onChange={e => setQuery(e.target.value)}
              autoFocus
            />
          </div>
          {genderBuckets.length > 1 && (
            <div style={{ display: 'flex', gap: 6, flexWrap: 'wrap', padding: '0 16px 8px', alignItems: 'center' }}>
              <span style={{ fontSize: 10, fontWeight: 700, opacity: 0.6 }}>성별</span>
              <button className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: genderFilter === 'all' ? 'var(--l-accent)' : 'var(--l-surface-2)', color: genderFilter === 'all' ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setGenderFilter('all')}>전체</button>
              {genderBuckets.map(g => (
                <button key={g.key} className="lovey-chip" style={{ cursor: 'pointer', border: 'none', background: genderFilter === g.key ? 'var(--l-accent)' : 'var(--l-surface-2)', color: genderFilter === g.key ? '#fff' : 'var(--l-ink-soft)' }} onClick={() => setGenderFilter(g.key)}>{g.label} <span style={{ opacity: 0.55 }}>{g.count}</span></button>
              ))}
            </div>
          )}
          <TagFilterBar groups={tagGroups} selected={selectedTags} onToggle={toggleTag} onClear={clearTags} chipClass="lovey-chip" accentVar="--l-accent" counts={tCounts} storageKey="lovey_tagcollapse" />
        </>
      )}

      <div className="lovey-scroll" ref={scrollRef}>
        {loading ? (
          <div className="lovey-grid">
            {Array.from({ length: 4 }).map((_, i) => (
              <div key={i} className="lovey-card">
                <div className="skeleton" style={{ width: '100%', aspectRatio: '3/4', borderRadius: 0 }} />
                <div className="lovey-card-body">
                  <div className="skeleton skeleton-line medium" />
                  <div className="skeleton skeleton-line short" />
                </div>
              </div>
            ))}
          </div>
        ) : visibleChars.length === 0 ? (
          selectedTags.length > 0 || query.trim()
            ? <div className="lovey-empty">검색 결과가 없습니다.</div>
          : view === 'favorites'
            ? <div className="lovey-empty">즐겨찾기한 캐릭터가 없습니다.<br />카드의 ★를 눌러 추가하세요.</div>
          : view === 'completed'
            ? <div className="lovey-empty">완결한 캐릭터가 없습니다.</div>
            : view === 'waiting'
              ? <div className="lovey-empty">대기 중인 캐릭터가 없습니다.</div>
              : items.length === 0
                ? <div className="lovey-empty">가져온 캐릭터가 없습니다<br />⋮ 메뉴에서 loveydovey.ai 캐릭터 URL로 가져오세요.<br />(메타데이터만 — 설정·도입부는 직접 입력)</div>
                : <div className="lovey-empty">진행 중인 캐릭터가 없습니다.</div>
        ) : (
          <VirtualCardGrid
            items={visibleChars}
            renderItem={renderCard}
            scrollRef={scrollRef}
            imageHeightRatio={4 / 3}
            bodyHeight={104}
            columns={2}
            gap={12}
            padX={16}
          />
        )}
      </div>
    </>
  )
}
```

- [ ] **Step 3: 타입체크/빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: 빌드 성공(타입 에러 없음).

- [ ] **Step 4: 전체 테스트 회귀 확인**

Run: `cd /home/server/StoryFit/apps/web && npm test`
Expected: 전체 PASS.

- [ ] **Step 5: 수동 검증 (수용 기준)**

개발 서버에서 loveydovey 리스트를 열고 확인:

1. **카운트 정확성** — 진행/대기/완결 숫자가 스크롤·검색·태그·성별 필터를 바꿔도 변하지 않고, DB 전체 총합과 일치(완결 탭으로 들어가 개수와 비교).
2. **가상 스크롤** — 수백~수천 개에서 스크롤이 매끄럽고, 새 항목이 위로 튀지 않음. devtools Elements에서 카드 DOM이 화면 분량(+오버스캔)만 존재.
3. **고정 높이 정렬** — 카드들이 행 단위로 깔끔히 정렬되고 겹침/빈틈이 없음. 어긋나면 `bodyHeight`(페이지 prop)와 `.lovey-card-body height`(CSS)를 같은 값으로 실측 보정.
4. **뒤로가기 복원** — 임의 위치까지 스크롤 → 카드 진입 → 브라우저 뒤로가기 → **직전 스크롤 위치·뷰 탭·검색/태그/성별 필터 그대로** 복원(스피너 없이 즉시).

- [ ] **Step 6: Commit**

```bash
cd /home/server/StoryFit/apps/web && git add app/\(loveydovey\)/loveydovey/page.tsx app/globals.css && git commit -m "feat: loveydovey 리스트 — useCenterList + 가상 스크롤 + 고정 높이 카드"
```

---

## After This Plan (out of scope, follow-up)

- **나머지 8개 센터 전파**: melting/tingle/babechat/tikita/rofan/zeta/whif/chub 페이지를 동일하게 `useCenterList` + `VirtualCardGrid`로 이주(각 센터 카드 마크업·CSS 고정 높이만 별도). whif는 `completed` 판정이 다를 수 있어 `viewCounts`의 `isDone` 커스터마이즈 필요 → 셀렉터에 옵션 추가 검토.
- **정리**: 전파 완료 후 `lib/useInfiniteScroll.ts`/`lib/useScrollRestore.ts` 제거 검토.
- **배포(2단계 push)**: `apps/web`(main) push → 부모 리포(master) 서브모듈 포인터 업데이트 push. 서버: `git pull origin master && git submodule update --remote apps/web && docker compose up --build -d`.
- **가이드 동기화**: 사용자 체감 변화(정확 카운트/가상 스크롤) 발생 시 `app/(main)/guide/page.tsx` `FEATURE_SECTIONS` 및 `context/adding-a-center.md` 갱신 검토.
```
