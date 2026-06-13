# 레이아웃 재설계 05 — 탐색 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 탐색을 외부 센터 전용 화면으로 재구성: 상단 센터 3개 + 각 센터별 "최근 등록" 가로 스와이프(최대 6 + 더보기 ›).

**Architecture:** 기존 explore의 "내 콘텐츠(캐릭터/페르소나)"를 제거하고, 센터별 최근 등록 항목을 가로 스크롤 레일로 보여준다. 항목 데이터는 각 센터 목록 API에서 최근 N개를 가져온다.

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01·02.

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인. 참고 목업: `content/explore-v3.html`.

---

## File Structure
- 수정: `apps/web/app/(main)/explore/page.tsx` — 센터 + 센터별 레일
- 수정: `apps/web/app/globals.css` — 탐색 레일/카드 스타일

> 데이터 소스 확인 필요: 각 센터(WHIF/ZETA/MELTING)의 "최근 등록 컬렉션/캐릭터" 목록을 주는 기존 API. 후보: `/api/collections?source=...` 또는 각 센터 목록 라우트. **Task 0에서 실제 엔드포인트를 확인**한 뒤 Task 2의 fetch URL을 확정한다.

---

### Task 0: 센터별 최근 항목 데이터 소스 확인

- [ ] **Step 1: 엔드포인트 조사**

Run: `cd apps/web && grep -rln "characterCollection\|/api/collections\|sourceUrl" app/api`
그리고 WHIF/ZETA/MELTING 목록 페이지(`app/(whif)/whif/page.tsx` 등)가 사용하는 API를 확인.
- 컬렉션 단위가 있으면 `/api/collections`(센터별 필터)에서 최근 6개.
- 없으면 `/api/characters`를 `collection.sourceUrl`/source로 그룹핑해 센터별 최근 6개 산출.

확정된 소스를 Task 2에 반영. (예: `api.get('/api/collections')` 후 클라에서 센터별 그룹핑)

---

### Task 1: 탐색 레일 컴포넌트 마크업

**Files:**
- Modify: `apps/web/app/(main)/explore/page.tsx`

- [ ] **Step 1: 페이지 구조 교체**

기존 CENTERS/SHORTCUTS 렌더를 아래로 교체. `SHORTCUTS`(내 캐릭터/페르소나) 제거. 센터 정의는 유지하되 desc/이모지 표시는 라벨용으로만.

```tsx
const CENTERS = [
  { key: 'WHIF', href: '/whif', grad: 'linear-gradient(135deg,#8b5cf6,#6d28d9)', railLabel: 'WHIF · 최근 세계관' },
  { key: 'ZETA', href: '/zeta', grad: 'linear-gradient(135deg,#7c5cff,#9d6bff)', railLabel: 'ZETA · 최근 플롯' },
  { key: 'MELTING', href: '/melting', grad: 'linear-gradient(135deg,#ff2e93,#ff5fae)', railLabel: 'MELTING · 최근 캐릭터' },
] as const
```

렌더:
```tsx
<div className="scroll explore-feed">
  <section className="home-sec">
    <div className="home-lbl">센터</div>
    <div className="home-centers">
      {CENTERS.map(c => (
        <button key={c.key} className="home-cen" style={{ background: c.grad }} onClick={() => router.push(c.href)}>{c.key}</button>
      ))}
    </div>
  </section>

  {CENTERS.map(c => (
    <section key={c.key} className="home-sec" style={{ ['--ac' as any]: CENTER_COLOR[c.key] }}>
      <div className="home-lbl">{c.railLabel}<button className="home-more" onClick={() => router.push(c.href)}>더보기 ›</button></div>
      <div className="ex-rail">
        {(recent[c.key] ?? []).map(it => (
          <button key={it.id} className="ex-it" onClick={() => router.push(it.href)}>
            {it.image ? <img src={it.image} alt="" /> : <div className="ex-noimg">🎭</div>}
            <div className="ex-bd">
              <span className="ex-ty" style={{ background: CENTER_COLOR[c.key] }}>{it.type}</span>
              <div className="ex-nm">{it.name}</div>
              <div className="ex-ds">{it.desc}</div>
            </div>
          </button>
        ))}
        <button className="ex-it more" onClick={() => router.push(c.href)}><div className="ex-morebox">›<br/>더보기</div></button>
      </div>
    </section>
  ))}
</div>
```
`CENTER_COLOR`는 `@/lib/convDisplay`에서 import.

- [ ] **Step 2: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: `recent` 타입 정의 필요 → Task 2.

---

### Task 2: 센터별 최근 항목 로딩

**Files:**
- Modify: `apps/web/app/(main)/explore/page.tsx`

- [ ] **Step 1: state·로딩 추가** (Task 0에서 확정한 API 사용)

```tsx
type ExItem = { id: string; name: string; type: string; desc: string; image?: string; href: string }
const [recent, setRecent] = useState<Record<string, ExItem[]>>({ WHIF: [], ZETA: [], MELTING: [] })

useEffect(() => {
  api.get('/api/collections').then((cols: any[]) => {
    const byCenter: Record<string, ExItem[]> = { WHIF: [], ZETA: [], MELTING: [] }
    for (const col of cols ?? []) {
      const center = getCenter(col.sourceUrl)        // lib/convDisplay
      if (!center) continue
      byCenter[center].push({
        id: col.id, name: col.title, type: '컬렉션',
        desc: col.description ?? '', image: col.coverImageUrl || undefined,
        href: `/whif/universes/${col.id}`,           // Task 0에서 센터별 상세 경로 확정
      })
    }
    for (const k of Object.keys(byCenter)) byCenter[k] = byCenter[k].slice(0, 6)
    setRecent(byCenter)
  }).catch(() => {})
}, [])
```
> Task 0 결과에 따라 엔드포인트·필드·상세 href를 실제 값으로 확정한다. 컬렉션이 없는 센터는 캐릭터 목록으로 대체.

- [ ] **Step 2: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 3: 탐색 레일 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가**

```css
.explore-feed{ padding:13px; display:flex; flex-direction:column; gap:17px; }
.ex-rail{ display:flex; gap:9px; overflow-x:auto; padding-bottom:3px; scrollbar-width:none; }
.ex-rail::-webkit-scrollbar{ display:none; }
.ex-it{ width:112px; flex-shrink:0; background:var(--paper); border:1px solid var(--hairline); border-radius:11px; overflow:hidden; display:flex; flex-direction:column; text-align:left; cursor:pointer; padding:0; color:var(--ink); }
.ex-it img{ width:100%; height:96px; object-fit:cover; display:block; }
.ex-it .ex-noimg{ width:100%; height:96px; display:grid; place-items:center; font-size:26px; background:var(--bg-2); }
.ex-bd{ padding:7px 8px 9px; display:flex; flex-direction:column; gap:2px; }
.ex-ty{ font-size:8px; font-weight:700; color:#fff; align-self:flex-start; padding:1px 6px; border-radius:20px; }
.ex-nm{ font-size:10.5px; font-weight:800; color:var(--ink); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.ex-ds{ font-size:9px; color:var(--ink-soft); line-height:1.4; margin-top:2px; display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden; min-height:25px; }
.ex-it.more{ width:64px; }
.ex-morebox{ width:64px; min-height:180px; border-radius:11px; border:1px dashed var(--chrome-border); display:flex; flex-direction:column; align-items:center; justify-content:center; gap:4px; color:var(--ink-soft); font-size:9px; font-weight:700; text-align:center; }
```

- [ ] **Step 2: 수동 확인**

`npm run dev` → /explore: 센터 3개 + 각 센터별 가로 스와이프(최근 6 + 더보기), 카드에 타입 배지·이름·설명 2줄. 내 캐릭터/페르소나 섹션 없음.

- [ ] **Step 3: 커밋**

```bash
git add "apps/web/app/(main)/explore/page.tsx" apps/web/app/globals.css
git commit -m "Feat: 탐색을 센터별 최근 등록 가로 스와이프로 재구성"
```

---

## Self-Review
- 스펙 탐색(외부 센터 전용 + 센터별 최근 레일) → Task 1~3. 데이터 소스는 Task 0에서 실코드 확인 후 확정(가정 최소화).
- 유실 방지: 내 캐릭터·페르소나는 홈 캐릭터 액션으로 이미 접근(스펙 명시) → 탐색에서 제거해도 유실 아님.
