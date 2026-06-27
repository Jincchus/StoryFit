# 센터 태그 필터 접기 + 뷰연동 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 공용 `TagFilterBar`에 접기(기본 접힘·상태 기억)를 넣고, 9개 센터 리스트 페이지의 태그 필터를 뷰+성별+검색 연동으로 바꾸며, 센터 추가 가이드를 동기화한다.

**Architecture:** 접기는 공용 `TagFilterBar` 컴포넌트 한 곳에 추가해 모든 소비처(9개 센터 + 전체 센터)가 자동 적용. 각 센터 페이지는 `tagGroups`/`tCounts` 입력을 "뷰+성별+검색이 적용된 base(태그 제외)"로 교체. 가이드 md + CLAUDE.md 규칙으로 향후 전파.

**Tech Stack:** Next.js 14 App Router(client components), React, TypeScript, Vitest(순수 함수만).

설계 문서: `context/2026-06-27-center-tagfilter-collapse-viewaware-design.md`

## Global Constraints

- 접기 기본값 = **접힘(collapsed=true)**. `storageKey` 전달 시 localStorage에 마지막 상태 기억.
- localStorage 읽기는 hydration mismatch 방지를 위해 `useEffect`(마운트 후)에서만. 초기 state는 `true`.
- 센터 태그 base = 뷰 + 성별 + 검색 적용 후(단, selectedTags 자신은 제외). 전체 센터(`explore/all`)와 동일한 좁혀가기.
- 작업 디렉터리 `apps/web`. 커밋은 apps/web(서브모듈). 배포는 전체 완료 후 사용자 확인 하에.
- 검증: `npx tsc --noEmit` exit 0 + `npx vitest run`(기존 테스트 유지). UI 동작은 수동 확인.
- storagePrefix(접기 키 접두사): whif=`whif`, zeta=`zeta`, melting=`melting`, tikita=`tikita`, chub=`chub`, rofan=`rofan`, loveydovey=`lovey`, babechat=`bc`, tingle=`tg`, 전체센터=`all`.

---

### Task 1: TagFilterBar 접기 + storageKey

**Files:**
- Modify: `components/ui/TagFilterBar.tsx` (전체 컴포넌트)

**Interfaces:**
- Produces: `TagFilterBar`에 옵션 prop `storageKey?: string` 추가. 동작/시그니처의 나머지는 동일.

- [ ] **Step 1: 컴포넌트 교체** — 파일 전체를 아래로 교체:

```tsx
'use client'
import { useEffect, useState } from 'react'

export interface TagGroup { category: string; tags: string[] }

export default function TagFilterBar({ groups, selected, onToggle, onClear, chipClass, accentVar, counts, storageKey }: {
  groups: TagGroup[]
  selected: string[]
  onToggle: (tag: string) => void
  onClear: () => void
  chipClass: string
  accentVar: string
  counts?: Record<string, number>
  storageKey?: string
}) {
  // 기본 접힘. storageKey 있으면 마운트 후 localStorage에서 복원(hydration mismatch 방지).
  const [collapsed, setCollapsed] = useState(true)
  useEffect(() => {
    if (!storageKey) return
    const v = localStorage.getItem(storageKey)
    if (v !== null) setCollapsed(v === '1')
  }, [storageKey])

  const toggle = () => setCollapsed(c => {
    const next = !c
    if (storageKey) localStorage.setItem(storageKey, next ? '1' : '0')
    return next
  })

  if (groups.length === 0) return null

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8, padding: '0 16px 8px' }}>
      <div>
        <button className={chipClass} style={{ cursor: 'pointer', border: 'none' }} onClick={toggle}>
          🏷 태그{selected.length > 0 ? ` (${selected.length})` : ''} {collapsed ? '▾' : '▴'}
        </button>
      </div>
      {!collapsed && (
        <div style={{ display: 'flex', flexDirection: 'column', gap: 8, maxHeight: 220, overflowY: 'auto' }}>
          {selected.length > 0 && (
            <div>
              <button className={chipClass} style={{ cursor: 'pointer', border: 'none' }} onClick={onClear}>✕ 전체 해제</button>
            </div>
          )}
          {groups.map(g => (
            <div key={g.category} style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
              <div style={{ fontSize: 10, fontWeight: 700, opacity: 0.6 }}>{g.category}</div>
              <div style={{ display: 'flex', flexWrap: 'wrap', gap: 6 }}>
                {g.tags.map(tag => {
                  const active = selected.includes(tag)
                  return (
                    <button
                      key={tag}
                      className={chipClass}
                      style={{ cursor: 'pointer', border: 'none', background: active ? `var(${accentVar})` : undefined, color: active ? '#fff' : undefined }}
                      onClick={() => onToggle(tag)}
                    >#{tag}{counts?.[tag] ? <span style={{ marginLeft: 4, opacity: active ? 0.8 : 0.5 }}>{counts[tag]}</span> : null}</button>
                  )
                })}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 2: 타입체크**

Run: `npx tsc --noEmit`
Expected: exit 0. (기존 호출부는 `storageKey` 미전달이라 옵셔널로 호환.)

- [ ] **Step 3: 커밋**

```bash
git add components/ui/TagFilterBar.tsx
git commit -m "feat: TagFilterBar 접기(기본 접힘·storageKey 상태 기억)"
```

---

### Task 2: 9개 센터 + 전체 센터 — 뷰연동 태그 base + storageKey

**목표:** 각 센터 리스트 페이지에서 `tagGroups`/`tCounts`의 입력을 "뷰+성별+검색이 적용된 base(selectedTags 제외)"로 바꾸고, `<TagFilterBar>`에 `storageKey`를 전달한다. 전체 센터는 base가 이미 연동돼 있으므로 `storageKey`만 추가한다.

**공통 패턴 (각 페이지에 맞게 적용):**
현재 각 센터는 `tagGroups = buildTagGroups(<전체리스트>.flatMap(c => c.tags ?? []), tagConfig)`, `tCounts = tagCounts(<전체리스트>)`를 쓴다. 이를 다음으로 바꾼다(페이지가 이미 가진 술어 재사용):

```ts
// 뷰+성별+검색 적용, 태그 필터는 제외한 base
const tagBase = <기준리스트>.filter(c =>
  <뷰술어(c)> && <성별술어(c)> && <검색술어(c)>
)
const tagGroups = buildTagGroups(tagBase.flatMap(c => c.tags ?? []), tagConfig)
const tCounts = tagCounts(tagBase)
```

그리고 `<TagFilterBar ... />`에 `storageKey="<prefix>_tagcollapse"`를 추가한다. 렌더용 `visible` 목록은 그대로 두되(이미 뷰/성별/검색/태그/정렬 모두 적용), 가능하면 `tagBase.filter(<태그술어>)`를 재사용해 중복 술어를 줄인다.

**기준 리스트 / 술어 표** (각 파일에서 실제 변수명을 확인해 매핑):

| 센터 | 파일 | 기준리스트 | 뷰술어 | 성별술어 | 검색술어 | storageKey |
|---|---|---|---|---|---|---|
| zeta | `app/(zeta)/zeta/page.tsx` | `plots` | view active/waiting/completed/favorites (started/completed/isFav) | `genderFilter==='all' \|\| cardGenderBucket(p.characters)===genderFilter` | `matchesQuery(p.title, p.tags)` | `zeta_tagcollapse` |
| melting | `app/(melting)/melting/page.tsx` | `chars` | 동일 | `matchesGender(c)` (기존 함수) | `matchesQuery(c.title, c.tags)` | `melting_tagcollapse` |
| tikita | `app/(tikita)/tikita/page.tsx` | `stories` | 동일 | `genderFilter==='all' \|\| cardGenderBucket(s.characters)===genderFilter` | `matchesQuery(s.title, s.tags)` | `tikita_tagcollapse` |
| chub | `app/(chub)/chub/page.tsx` | `chars` | 동일 | gender 인라인 비교 | `matchesQuery(c.title, c.tags)` | `chub_tagcollapse` |
| rofan | `app/(rofan)/rofan/page.tsx` | `chars` | 동일 | gender 인라인 | `matchesQuery(c.title, c.tags)` | `rofan_tagcollapse` |
| loveydovey | `app/(loveydovey)/loveydovey/page.tsx` | `chars` | 동일 | gender 인라인 | `matchesQuery(c.title, c.tags)` | `lovey_tagcollapse` |
| babechat | `app/(babechat)/babechat/page.tsx` | `chars` | 동일 | gender 인라인 | `matchesQuery(c.title, c.tags)` | `bc_tagcollapse` |
| tingle | `app/(tingle)/tingle/page.tsx` | `colsByType` (타입탭 적용 후) | view 술어 | `typeTab!=='character' \|\| genderFilter==='all' \|\| cardGenderBucket(c.characters)===genderFilter` (기존 분기 보존) | `matchesQuery(c)` | `tg_tagcollapse` |
| whif | `app/(whif)/whif/page.tsx` | `tab==='universes' ? universes : characters` | tab별 view 술어 | characters 탭에서만 gender | `matchesQuery(item.title, item.tags)` | `whif_tagcollapse` |
| 전체센터 | `app/(main)/explore/all/page.tsx` | (base 이미 연동) | — | — | — | `all_tagcollapse` |

> 각 파일의 실제 뷰/성별/검색 술어는 기존 `visible`/`filtered` 계산식에서 그대로 가져온다. tingle·whif는 타입/탭 분기를 반드시 보존한다.

**Files (수정):** 위 표의 10개 파일.

- [ ] **Step 1: 센터별 적용 (파일 1개씩, 커밋도 1개씩)** — 각 센터 페이지에서:
  1. 해당 파일의 현재 `tagGroups`/`tCounts` 정의와 `visible`/`filtered` 필터 체인을 읽는다.
  2. `tagBase`를 위 패턴으로 추가(뷰+성별+검색, 태그 제외)하고 `tagGroups`/`tCounts` 입력을 `tagBase`로 교체.
  3. `<TagFilterBar ... storageKey="<prefix>_tagcollapse" />` 추가.
  4. `npx tsc --noEmit` → exit 0 확인 후 커밋:

```bash
git add "<해당 파일>"
git commit -m "feat(<center>): 태그 필터 뷰+성별+검색 연동 + 접기 storageKey"
```

- [ ] **Step 2: 전체 센터 storageKey** — `app/(main)/explore/all/page.tsx`의 `<TagFilterBar ... />`에 `storageKey="all_tagcollapse"` 추가. `npx tsc --noEmit` 후 커밋.

- [ ] **Step 3: 전체 타입체크 + 기존 테스트**

Run: `npx tsc --noEmit && npx vitest run`
Expected: exit 0, 테스트 전부 PASS.

- [ ] **Step 4: 수동 검증(센터 1곳)** — 개발 서버에서: 진행중 탭 → 태그 목록이 진행중 카드 태그만인지, 완결 탭 전환 시 완결 카드 태그로 바뀌는지; 태그 바가 기본 접힘이고 펼친 뒤 새로고침해도 펼침이 유지되는지(센터별 독립).

---

### Task 3: 가이드 + CLAUDE.md 동기화

**Files:**
- Modify: `context/adding-a-center.md` (section E + 검증 체크리스트)
- Modify: `CLAUDE.md` (Rules)

- [ ] **Step 1: adding-a-center.md — section E에 항목 추가** — "E. 탐색 / 목록 부가 표시"의 마지막 줄(`admin/center-tags` 위) 근처에 추가:

```markdown
- **리스트 페이지 검색/필터/태그(필수 패턴)**: 공용 `components/ui/TagFilterBar`를 쓰고
  `storageKey="<storagePrefix>_tagcollapse"`를 전달한다(접기 상태 센터별 기억). 태그 목록·카운트
  (`buildTagGroups`/`tagCounts`)는 **뷰+성별+검색이 적용된 base(태그 자신 제외)** 로 계산해
  진행중 탭이면 진행중 카드 태그만 보이게 한다(전체 센터·타 센터와 동일).
```

- [ ] **Step 2: adding-a-center.md — 검증 체크리스트에 항목 추가** — "## 3. 작업 후 검증 체크리스트"에 추가:

```markdown
- [ ] 뷰탭(진행/대기/완결) 전환 시 해당 상태 카드의 태그만 노출되는지
- [ ] 태그 필터바가 기본 접힘이고, 펼친 상태가 새로고침 후에도 유지되는지(센터별 독립)
```

- [ ] **Step 3: CLAUDE.md — Rules에 규칙 추가** — `## Rules (always follow)` 섹션의 마지막 규칙 뒤에 추가:

```markdown
- **센터 공통 동작 ↔ 가이드 동기화**: 센터 공통 검색/필터/태그/페르소나 등 동작을 추가·변경하면 `context/adding-a-center.md`도 함께 갱신해, 새 센터 추가 시 누락 없이 전파되도록 한다.
```

- [ ] **Step 4: 커밋**

```bash
git add context/adding-a-center.md CLAUDE.md
git commit -m "docs: 센터 검색/필터/태그 패턴 가이드 + CLAUDE.md 동기화 규칙"
```

> 참고: `CLAUDE.md`와 `context/adding-a-center.md`는 부모 레포(StoryFit) 루트에 있다(서브모듈 apps/web가 아님). 이 커밋은 부모 레포에서 수행한다.

---

## 배포 (전체 완료 후, 사용자 확인 하에)

```bash
# apps/web (TagFilterBar + 페이지들)
cd apps/web && git push origin HEAD:main
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (태그 필터 접기 + 뷰연동)" && git push origin master
# CLAUDE.md/adding-a-center.md는 부모 레포 커밋에 이미 포함
```

## Self-Review 메모

- 스펙 커버리지: 접기(Task 1), 뷰연동 태그 base + storageKey(Task 2, 9센터+전체센터), 가이드+CLAUDE.md(Task 3) — 전부 매핑.
- TagFilterBar는 React 컴포넌트라 단위 테스트 없음 → tsc + 수동. 순수 로직 변경이 없어 신규 vitest 불필요(기존 144 유지).
- ⚠️ Task 2는 9개 파일 구조가 달라(tingle 타입, whif 탭) 실제 현재 코드의 술어를 읽고 base를 정확히 분리할 것. 커밋은 센터별로.
