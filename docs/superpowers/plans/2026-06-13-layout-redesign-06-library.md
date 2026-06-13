# 레이아웃 재설계 06 — 서재 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 서재를 홈 완결과 동일한 카드 그리드로 재구성하고, 카드 ⋯ 메뉴(꺼내기/소설 내보내기/삭제)를 제공한다.

**Architecture:** 기존 `library/page.tsx`의 데이터 로딩(`/api/library`)·꺼내기(unarchive)는 유지하고, 행 리스트를 2열 카드 그리드로 교체한다. 카드 ⋯ 메뉴에 꺼내기·소설 내보내기(`/api/conversations/:id/export`)·삭제를 모은다.

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01·02·04(convDisplay).

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인. 참고 목업: `content/library.html`.

---

## File Structure
- 수정: `apps/web/app/(main)/library/page.tsx` — 카드 그리드 + ⋯ 메뉴
- 수정: `apps/web/app/globals.css` — 서재 카드/메뉴 스타일(홈 `home-card`/`home-grid` 재사용 + `lib-menu`)

> 소설 내보내기 엔드포인트 확인: Task 0.

---

### Task 0: 소설 내보내기·삭제 경로 확인

- [ ] **Step 1: 경로 조사**

Run: `cd apps/web && grep -rln "export" app/api/conversations`
완결작 소설 내보내기 라우트(예: `/api/conversations/:id/export` → .txt) 경로 확인. 삭제는 기존 `DELETE /api/conversations/:id`. 확정 경로를 Task 2에 반영.

---

### Task 1: 서재 카드 그리드 마크업

**Files:**
- Modify: `apps/web/app/(main)/library/page.tsx`

- [ ] **Step 1: import + 헤더/필터**

```tsx
import { getCenter, CENTER_COLOR, MODE_LABEL, convTitleParts } from '@/lib/convDisplay'
```
헤더: "서재 / 완결된 이야기 N편" 유지. (정렬·모드 필터는 선택 — 우선 카드 그리드만, 필터는 후속 가능)

- [ ] **Step 2: 리스트 → 카드 그리드 교체**

기존 `.row` 리스트를 아래로 교체. `ConvItem`에 `sourceUrl?`, `mode`, `personaCharacter?` 포함하도록 보강.

```tsx
<div className="home-grid">
  {conversations.map(conv => {
    const t = convTitleParts(conv); const center = getCenter(conv.sourceUrl); const char = conv.characters[0]?.character
    return (
      <div key={conv.id} className="home-card lib-card">
        <button className="lib-cardmain" onClick={() => router.push(`/library/${conv.id}`)}>
          <div className="hc-top">
            <button className="lib-dots" aria-label="메뉴" onClick={e => { e.stopPropagation(); setMenuId(m => m === conv.id ? null : conv.id) }}>⋯</button>
            {center && <span className="hc-center" style={{ background: CENTER_COLOR[center] }}>{center}</span>}
            {char?.avatarUrl ? <img src={char.avatarUrl} alt="" /> : <div className="hc-noimg">🎭</div>}
          </div>
          <div className="hc-bd">
            <div className="hc-t">{t.room}<span className="hc-pp">({t.chars}{t.persona ? `.${t.persona}` : ''})</span></div>
            <div className="hc-q">{previewText(conv.messages[0]?.content ?? '')}</div>
            <div className="hc-ft"><span className="md">{MODE_LABEL[conv.mode] ?? conv.mode}</span> · {new Date(conv.updatedAt).toLocaleDateString('ko-KR')}</div>
          </div>
        </button>
        {menuId === conv.id && (
          <>
            <div style={{ position: 'fixed', inset: 0, zIndex: 19 }} onClick={() => setMenuId(null)} />
            <div className="lib-menu">
              <button onClick={() => { setMenuId(null); setUnarchiveId(conv.id) }}>↩ 꺼내기</button>
              <button onClick={() => { setMenuId(null); downloadNovel(conv.id) }}>📄 소설 내보내기</button>
              <button className="danger" onClick={() => { setMenuId(null); setDeleteId(conv.id) }}>🗑 삭제</button>
            </div>
          </>
        )}
      </div>
    )
  })}
</div>
```

- [ ] **Step 3: state·핸들러 추가**

```tsx
const [menuId, setMenuId] = useState<string | null>(null)
const [deleteId, setDeleteId] = useState<string | null>(null)
const downloadNovel = async (id: string) => {
  const res = await fetch(`/api/conversations/${id}/export`, { credentials: 'include' })  // Task 0 경로
  const blob = await res.blob()
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'story.txt'; a.click(); URL.revokeObjectURL(a.href)
}
const handleDelete = async (id: string) => { await api.delete(`/api/conversations/${id}`); setConversations(p => p.filter(c => c.id !== id)); setDeleteId(null) }
```
삭제 확인용 ConfirmDialog 추가(기존 unarchive ConfirmDialog 패턴 재사용).

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 2: 서재 메뉴 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가** (home-card/home-grid는 Plan 03에서 정의됨)

```css
.lib-card{ position:relative; }
.lib-cardmain{ all:unset; display:flex; flex-direction:column; cursor:pointer; }
.lib-dots{ position:absolute; top:5px; left:6px; z-index:2; width:20px; height:20px; border-radius:50%; background:rgba(0,0,0,.6); color:#fff; border:none; font-size:12px; display:grid; place-items:center; cursor:pointer; }
.lib-menu{ position:absolute; top:28px; left:8px; z-index:20; width:124px; background:var(--chrome-face); border:1px solid var(--chrome-border); border-radius:10px; overflow:hidden; box-shadow:0 8px 22px rgba(0,0,0,.45); }
.lib-menu button{ display:block; width:100%; text-align:left; padding:9px 11px; font-size:11px; color:var(--ink); background:none; border:none; border-bottom:1px solid var(--hairline); cursor:pointer; }
.lib-menu button:last-child{ border-bottom:none; }
.lib-menu button.danger{ color:#ff7a90; }
```

- [ ] **Step 2: 수동 확인**

`npm run dev` → /library: 완결작이 2열 카드(사진·센터배지·`방명(캐릭터.페르소나)`·모드·완결일), ⋯ 메뉴에서 꺼내기/소설 내보내기/삭제 동작, 카드 클릭 시 읽기 화면.

- [ ] **Step 3: 커밋**

```bash
git add "apps/web/app/(main)/library/page.tsx" apps/web/app/globals.css
git commit -m "Feat: 서재 완결작 카드 그리드 + ⋯ 메뉴(꺼내기/내보내기/삭제)"
```

---

## Self-Review
- 스펙 서재(완결 카드 그리드 + ⋯ 메뉴) → Task 1~2. 내보내기·삭제 경로는 Task 0 확인 후 확정.
- 유실 방지: 꺼내기(unarchive) 유지, 읽기 화면(`/library/:id`) 진입 유지, 소설 내보내기 진입을 카드 메뉴로 보존.
