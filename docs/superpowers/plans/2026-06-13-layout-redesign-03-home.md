# 레이아웃 재설계 03 — 홈 재구성 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 홈을 단일 피드로 재구성: 이어하기(1) → 센터(WHIF/ZETA/MELTING) → 액션 4개 → 대화중(최대 4) → 완결(서재 최근).

**Architecture:** 기존 홈의 온보딩/가져오기 모달은 유지하고 본문 섹션만 교체한다. 데이터는 `/api/conversations`(이어하기·대화중)와 `/api/library`(완결)에서 가져온다. 제목은 `채팅방명(캐릭터.페르소나)` 헬퍼로 조합한다.

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01·02.

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인. (apps/web 기준)

---

## File Structure
- 수정: `apps/web/app/(main)/page.tsx` — 홈 본문 섹션 교체(모달·가져오기 로직 유지)
- 수정: `apps/web/app/globals.css` — 홈 카드/센터/액션 스타일(`home-*` 클래스)

참고 목업: `.superpowers/brainstorm/<session>/content/home-v7.html`

---

### Task 1: 공통 헬퍼 — 제목·센터·시간

**Files:**
- Modify: `apps/web/app/(main)/page.tsx`

- [ ] **Step 1: 헬퍼 추가**

`page.tsx` 상단(컴포넌트 밖)에 추가. `timeAgo`는 기존 것 유지, 아래 2개 신규:

```tsx
function getCenter(sourceUrl?: string): 'WHIF' | 'ZETA' | 'MELTING' | null {
  if (!sourceUrl) return null
  if (sourceUrl.includes('zeta-ai.io')) return 'ZETA'
  if (sourceUrl.includes('melting.chat')) return 'MELTING'
  if (sourceUrl.includes('whif.')) return 'WHIF'
  return null
}

// 제목: "채팅방명(캐릭터[,추가…].페르소나)" — 캐릭터 다수면 … 처리
function convTitle(c: {
  title: string
  characters: { character: { name: string } }[]
  personaCharacter?: { name: string } | null
}): { room: string; chars: string; persona: string } {
  const names = c.characters.map(x => x.character.name)
  const chars = names.length <= 1 ? (names[0] ?? '') : `${names[0]},${names[1]}${names.length > 2 ? '…' : ''}`
  return { room: c.title, chars, persona: c.personaCharacter?.name ?? '' }
}

const CENTER_COLOR: Record<string, string> = { WHIF: '#8b5cf6', ZETA: '#7c5cff', MELTING: '#ff2e93' }
const MODE_LABEL: Record<string, string> = { story: '스토리', multiStory: '멀티' }
```

- [ ] **Step 2: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: 헬퍼 에러 없음(아직 미사용 경고 가능).

---

### Task 2: 데이터 로딩 (대화중 + 완결)

**Files:**
- Modify: `apps/web/app/(main)/page.tsx`

- [ ] **Step 1: state·로딩 교체**

기존 `recent`/`recentLoading` 대신:

```tsx
const [convs, setConvs] = useState<RecentConv[]>([])
const [completed, setCompleted] = useState<RecentConv[]>([])
const [loading, setLoading] = useState(true)
```

`RecentConv` 인터페이스에 `sourceUrl?: string`, `mode: string`, `isAutoCreated?: boolean`, `personaCharacter?: { name: string } | null` 포함하도록 보강.

useEffect 로딩:
```tsx
useEffect(() => {
  setIsAdmin(getIsAdmin())
  if (!localStorage.getItem('sf_onboarded')) setShowGuide(true)
  Promise.all([
    api.get('/api/conversations').catch(() => []),
    api.get('/api/library').catch(() => []),
  ]).then(([all, lib]: [RecentConv[], RecentConv[]]) => {
    const sorted = [...(all ?? [])].sort((a, b) => new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime())
    setConvs(sorted)
    setCompleted([...(lib ?? [])].sort((a, b) => new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime()))
  }).finally(() => setLoading(false))
}, [])
```

파생값:
```tsx
const ongoing = convs.filter(c => !c.isAutoCreated)
const latest = ongoing[0] ?? null
const ongoingCards = ongoing.slice(0, 4)
const completedCards = completed.slice(0, 4)
```

- [ ] **Step 2: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: 에러 없음.

---

### Task 3: 본문 섹션 렌더 교체

**Files:**
- Modify: `apps/web/app/(main)/page.tsx`

- [ ] **Step 1: `.scroll` 본문(219-302행 영역)을 새 섹션으로 교체**

기존 "새 대화 버튼 + 이어하기(3) + 바로가기 그리드"를 아래 구조로 교체. 모달(showGuide/showImport)과 떠있는 `?` 버튼 관련 부분은 ⚠ — `?` 플로팅 버튼은 제거(가이드는 액션으로). 온보딩/가져오기 모달 JSX는 유지.

카드 렌더 헬퍼(컴포넌트 내부 함수):
```tsx
const Card = (c: RecentConv) => {
  const t = convTitle(c)
  const center = getCenter(c.sourceUrl)
  const char = c.characters[0]?.character
  return (
    <button key={c.id} className="home-card" onClick={() => router.push(`/conversations/${c.id}`)}>
      <div className="hc-top">
        {center && <span className="hc-center" style={{ background: CENTER_COLOR[center] }}>{center}</span>}
        {char?.avatarUrl ? <img src={char.avatarUrl} alt="" /> : <div className="hc-noimg">🎭</div>}
      </div>
      <div className="hc-bd">
        <div className="hc-t">{t.room}<span className="hc-pp">({t.chars}{t.persona ? `.${t.persona}` : ''})</span></div>
        <div className="hc-q">{c.messages?.[0]?.content ? previewText(c.messages[0].content) : ''}</div>
        <div className="hc-ft"><span className="md">{MODE_LABEL[c.mode] ?? c.mode}</span> · {timeAgo(c.updatedAt)}</div>
      </div>
    </button>
  )
}
```

본문:
```tsx
<div className="scroll home-feed">
  {/* 이어하기 */}
  {latest && (() => {
    const t = convTitle(latest); const center = getCenter(latest.sourceUrl); const char = latest.characters[0]?.character
    return (
      <section className="home-sec">
        <div className="home-lbl">이어하기</div>
        <button className="home-hero" onClick={() => router.push(`/conversations/${latest.id}`)}>
          {center && <span className="hc-center hero-center" style={{ background: CENTER_COLOR[center] }}>{center}</span>}
          {char?.avatarUrl ? <img className="hero-cov" src={char.avatarUrl} alt="" /> : <div className="hero-cov hc-noimg">🎭</div>}
          <div className="hero-meta">
            <div className="hc-t">{t.room}<span className="hc-pp">({t.chars}{t.persona ? `.${t.persona}` : ''})</span></div>
            <div className="hero-q">{latest.messages?.[0]?.content ? previewText(latest.messages[0].content) : ''}</div>
            <div className="hero-time">{timeAgo(latest.updatedAt)}</div>
          </div>
        </button>
      </section>
    )
  })()}

  {/* 센터 */}
  <section className="home-sec">
    <div className="home-lbl">센터</div>
    <div className="home-centers">
      <button className="home-cen" style={{ background: 'linear-gradient(135deg,#8b5cf6,#6d28d9)' }} onClick={() => router.push('/whif')}>WHIF</button>
      <button className="home-cen" style={{ background: 'linear-gradient(135deg,#7c5cff,#9d6bff)' }} onClick={() => router.push('/zeta')}>ZETA</button>
      <button className="home-cen" style={{ background: 'linear-gradient(135deg,#ff2e93,#ff5fae)' }} onClick={() => router.push('/melting')}>MELTING</button>
    </div>
  </section>

  {/* 액션 4개 */}
  <section className="home-sec">
    <div className="home-acts">
      <button className="home-act" onClick={() => router.push('/guide')}><span className="ic">📖</span>가이드</button>
      <button className="home-act primary" onClick={() => router.push('/conversations/new')}><span className="ic">✨</span>새 대화</button>
      <button className="home-act" onClick={() => router.push('/characters')}><span className="ic">🎭</span>캐릭터</button>
      <button className="home-act" onClick={() => setShowImport(true)}><span className="ic">📥</span>가져오기</button>
    </div>
  </section>

  {/* 대화중 */}
  {ongoingCards.length > 0 && (
    <section className="home-sec">
      <div className="home-lbl">대화중 <button className="home-more" onClick={() => router.push('/chatlist')}>전체 ›</button></div>
      <div className="home-grid">{ongoingCards.map(Card)}</div>
    </section>
  )}

  {/* 완결 */}
  {completedCards.length > 0 && (
    <section className="home-sec">
      <div className="home-lbl">완결 <button className="home-more" onClick={() => router.push('/library')}>서재 ›</button></div>
      <div className="home-grid">{completedCards.map(Card)}</div>
    </section>
  )}
</div>
```

- [ ] **Step 2: 떠있는 `?` 버튼 제거 + shortcuts 배열 제거**

기존 `shortcuts` 배열과 하단 플로팅 `?` 버튼 블록 삭제. `isAdmin`은 더 이상 홈에서 안 쓰면(관리자 진입은 계정 메뉴) `getIsAdmin`/`isAdmin` 제거. `handleLogout`도 제거(계정 메뉴로 이동).

- [ ] **Step 3: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 4: 홈 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가** (목업 home-v7 기준)

```css
.home-feed{ padding:13px; display:flex; flex-direction:column; gap:16px; }
.home-sec{ display:flex; flex-direction:column; gap:9px; }
.home-lbl{ font-size:12px; font-weight:800; color:var(--ink); display:flex; align-items:center; gap:6px; }
.home-lbl::before{ content:''; width:3.5px; height:13px; background:var(--accent); border-radius:2px; }
.home-more{ margin-left:auto; font-size:10px; font-weight:600; color:var(--ink-soft); background:none; border:none; cursor:pointer; }
.hc-center{ font-size:8px; font-weight:800; color:#fff; padding:2px 6px; border-radius:20px; }

.home-hero{ position:relative; display:flex; gap:11px; align-items:stretch; text-align:left; cursor:pointer;
  border-radius:13px; background:linear-gradient(135deg,#271d3a,#191920); border:1px solid #38335180; padding:13px; color:var(--ink); }
.hero-center{ position:absolute; top:10px; right:11px; }
.hero-cov{ width:52px; align-self:stretch; height:auto; object-fit:cover; border-radius:7px; flex-shrink:0; border:1px solid #4a3f6a; }
.hero-meta{ flex:1; min-width:0; display:flex; flex-direction:column; }
.hero-q{ font-size:11.5px; color:#b7b0cc; margin-top:5px; line-height:1.4; display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden; }
.hero-time{ margin-top:auto; align-self:flex-end; font-size:10px; color:#7e7888; padding-top:8px; }

.home-centers{ display:flex; gap:8px; }
.home-cen{ flex:1; border-radius:11px; height:54px; color:#fff; font-size:13px; font-weight:800; letter-spacing:.5px; border:none; cursor:pointer; }

.home-acts{ display:grid; grid-template-columns:repeat(4,1fr); gap:8px; }
.home-act{ background:var(--pane); border:1px solid var(--hairline); border-radius:11px; padding:11px 4px; display:flex; flex-direction:column; gap:5px; align-items:center; font-size:9.5px; font-weight:700; color:var(--ink-2); cursor:pointer; }
.home-act .ic{ font-size:19px; }
.home-act.primary{ background:linear-gradient(135deg,#8b5cf6,#6d28d9); border-color:#8b5cf6; color:#fff; }

.home-grid{ display:grid; grid-template-columns:1fr 1fr; gap:9px; }
.home-card{ background:var(--paper); border:1px solid var(--hairline); border-radius:12px; overflow:hidden; display:flex; flex-direction:column; text-align:left; cursor:pointer; color:var(--ink); padding:0; }
.hc-top{ height:84px; position:relative; }
.hc-top img{ width:100%; height:100%; object-fit:cover; display:block; }
.hc-top .hc-noimg{ width:100%; height:100%; display:grid; place-items:center; font-size:24px; background:var(--bg-2); }
.hc-top .hc-center{ position:absolute; top:6px; right:6px; z-index:1; }
.hc-bd{ padding:8px 9px 9px; flex:1; display:flex; flex-direction:column; }
.hc-t{ font-size:11px; font-weight:800; color:var(--ink); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.hc-pp{ color:var(--accent); font-weight:600; }
.hc-q{ font-size:9.5px; color:var(--ink-soft); margin-top:4px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.hc-ft{ font-size:8.5px; color:var(--ink-faint); margin-top:auto; padding-top:7px; text-align:right; }
.hc-ft .md{ color:var(--accent); font-weight:700; }
```

- [ ] **Step 2: 빌드·수동 확인**

Run: `cd apps/web && npx next build`
이후 `npm run dev`: 홈에 이어하기 1건(사진 꽉 채움)·센터 3개·액션 4개·대화중(최대 4)·완결(최대 4)이 순서대로 보이고, 제목이 `방명(캐릭터.페르소나)` 형식, 카드 우상단 센터 배지, 우하단 `모드·시간` 표시.

- [ ] **Step 3: 커밋**

```bash
git add "apps/web/app/(main)/page.tsx" apps/web/app/globals.css
git commit -m "Feat: 홈 단일 피드 재구성(이어하기·센터·액션·대화중·완결)"
```

---

## Self-Review
- 스펙 홈 섹션 5종(이어하기/센터/액션/대화중/완결) → Task 3 커버. 제목·센터·모드 헬퍼 Task 1.
- 유실 방지: 가져오기 모달·온보딩 모달 유지, 가이드/캐릭터/새 대화/가져오기 액션 보존. `?` 플로팅·로그아웃·관리자는 계정 메뉴로 이관(중복 제거).
- 플레이스홀더 없음. `RecentConv`에 sourceUrl/mode/isAutoCreated/personaCharacter 필드 보강 명시.
