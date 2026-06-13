# 레이아웃 재설계 07 — 채팅방 + 설정 패널 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 채팅방 헤더를 ☰/⚙ 2버튼으로 단순화(☰에 스탯·인벤·줄거리·통화)하고, 장/턴을 텍스트로, 상태줄을 분리한다. 유저 메시지를 E안(우측 세로바)으로 바꾸고, 설정 패널을 3탭(기본/기억/세계관)으로 재편한다. **유실 방지 체크리스트의 모든 기존 기능을 보존한다.**

**Architecture:** `conversations/[id]/page.tsx`의 기능 로직(스트림·커맨드·관전·재회·주사위·페이지네이션 등)은 유지하고 헤더 마크업과 ＋메뉴 일부, 메시지 정렬 스타일만 손본다. `SidePanel`의 기존 섹션들을 3개 탭으로 묶는다(섹션 내부 JSX는 유지).

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01(테마/배경 제거)·02(독 숨김). Plan 01에서 customBg/theme는 이미 제거됨.

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인(유실 방지 체크리스트 전수). 참고 목업: `content/chatroom-v4.html`, `chatroom-v3.html`, `settings-basic-v2/memory/world.html`, `char-card-modal-v2.html`.

---

## File Structure
- 수정: `apps/web/app/(main)/conversations/[id]/page.tsx` — 헤더(☰/⚙), 상태줄, ☰ 메뉴
- 수정: `apps/web/app/(main)/conversations/[id]/_components/MessageList.tsx` — 유저 메시지 E안
- 수정: `apps/web/app/(main)/conversations/[id]/_components/SidePanel.tsx` — 3탭 재편
- 수정: `apps/web/components/ui/CharacterCardModal.tsx` — 수정 연필 위치/닫기
- 수정: `apps/web/app/globals.css` — 헤더/상태줄/☰메뉴/유저메시지/탭 스타일

---

### Task 1: 채팅방 헤더 단순화(☰/⚙) + 장·턴 텍스트 + 상태줄 분리

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/page.tsx:711-792`

- [ ] **Step 1: 헤더 우측 아이콘 → ☰/⚙ 2개로 교체**

기존 우측 아이콘 묶음(🎒/STAT/📜/📞/⚙, 746-785행)을 ☰(메뉴 토글) + ⚙(패널)로 교체. 새 state `showHeaderMenu` 추가.

```tsx
<div className="hstack" style={{ flexShrink: 0, gap: 6 }}>
  <button className="cr-hb" aria-label="기능 메뉴" onClick={() => setShowHeaderMenu(v => !v)}>☰</button>
  <button className={`cr-hb ${showPanel ? 'on' : ''}`} aria-label="대화 설정" onClick={() => setShowPanel(p => !p)}>⚙</button>
</div>
```

- [ ] **Step 2: 이름줄/배지 → 이름·페르소나 + 텍스트 메타로 교체**

기존 이름줄(721-733행)의 mode-badge/chapter-badge를 제거하고, 이름 아래에 plain 텍스트 메타(`스토리 · 3장 · 턴 N`)를 둔다. 페르소나 `▾` 버튼(setShowPanel)은 유지.

```tsx
<div className="cr-sub">
  {isMulti ? '멀티' : '스토리'}
  {conv?.autoChapterEnabled && (conv.chapter ?? 1) > 1 ? ` · ${conv.chapter}장` : ''}
  {` · 턴 ${Math.floor(messages.length / 2)}`}
</div>
```
기존 턴/타임라인 줄(734-743행, `showTimelineFull` 토글)은 상태줄 박스로 이동(Step 3).

- [ ] **Step 3: 상태줄 별도 박스**

헤더 아래에 statusTimeline 박스(클릭 시 `showTimelineFull` 토글)를 둔다. 기존 `showTimelineFull` 전체 표시 블록(788-792행)은 유지하되 트리거를 이 박스로.

```tsx
{conv.statusTimeline && (
  <button className="cr-statusbar" onClick={() => setShowTimelineFull(v => !v)}>
    <span>🎬</span><span className="tx">{conv.statusTimeline}</span><span className="ar">{showTimelineFull ? '▴' : '▾'}</span>
  </button>
)}
```

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 2: ☰ 헤더 메뉴 (스탯/인벤토리/줄거리/통화)

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/page.tsx`

- [ ] **Step 1: ☰ 메뉴 렌더 추가**

헤더 근처에 `showHeaderMenu` 드롭다운. 각 항목은 **기존 토글/오버레이 핸들러를 그대로 호출**(StatsPopover=`setShowStats`, InventoryPopover=`setShowInventory`, Recap=`setShowRecap`+`loadRecap`, VoiceCall=`setShowVoiceCall`). 조건부 노출은 기존과 동일(스탯/인벤 enabled 등).

```tsx
{showHeaderMenu && (
  <>
    <div style={{ position:'fixed', inset:0, zIndex:9 }} onClick={() => setShowHeaderMenu(false)} />
    <div className="cr-hmenu">
      {isStoryOrMulti && conv.statsEnabled && conv.statsConfig?.length > 0 && (
        <button onClick={() => { setShowHeaderMenu(false); setShowStats(true); setShowInventory(false) }}><span className="ic">STAT</span>스탯</button>
      )}
      {isStoryOrMulti && conv.inventoryEnabled && (
        <button onClick={() => { setShowHeaderMenu(false); setShowInventory(true); setShowStats(false) }}><span className="ic">🎒</span>인벤토리</button>
      )}
      {isStoryOrMulti && (
        <button onClick={() => { setShowHeaderMenu(false); setShowRecap(true); loadRecap() }}><span className="ic">📜</span>줄거리</button>
      )}
      <button onClick={() => { setShowHeaderMenu(false); setShowVoiceCall(true) }}><span className="ic">📞</span>음성 통화</button>
    </div>
  </>
)}
```

- [ ] **Step 2: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0. 기존 StatsPopover/InventoryPopover/RecapPopover/VoiceCallOverlay 렌더부는 그대로 유지(변경 없음).

- [ ] **Step 3: 커밋**

```bash
git add "apps/web/app/(main)/conversations/[id]/page.tsx"
git commit -m "Feat: 채팅방 헤더 ☰/⚙ 단순화 + 상태줄 분리 + ☰ 메뉴(스탯/인벤/줄거리/통화)"
```

---

### Task 3: 유저 메시지 E안(우측 세로바)

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/_components/MessageList.tsx`

- [ ] **Step 1: 유저 메시지 컨테이너 클래스/스타일 변경**

유저(`role === 'user'`) 메시지 렌더에서 기존 정렬을 E안으로: 우측 정렬 + 오른쪽 보라 세로바. 말풍선 배경 제거. 주사위 배지(`.dice-badge`)·편집폼은 유지. 해당 메시지 컨테이너에 `cr-user-e` 클래스 부여(또는 기존 클래스 교체).

```tsx
// 유저 메시지 wrapper
<div className="cr-user-e">{/* 기존 내용(텍스트/주사위 배지/편집) 유지 */}</div>
```

- [ ] **Step 2: 메시지 탭 액션 보존 확인**

재생성/편집/분기/삭제/북마크/음성재생(`speak`) 액션(345-367행)이 그대로 동작하는지 확인 — 마크업 변경은 정렬 wrapper만, 액션 버튼 로직은 건드리지 않는다.

- [ ] **Step 3: 빌드 검증**

Run: `cd apps/web && npx next build`
Expected: 성공.

- [ ] **Step 4: 커밋**

```bash
git add "apps/web/app/(main)/conversations/[id]/_components/MessageList.tsx"
git commit -m "Feat: 유저 메시지 E안(우측 세로바) 적용"
```

---

### Task 4: 설정 패널 3탭 재편

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/_components/SidePanel.tsx`

- [ ] **Step 1: 탭 state + 탭 바 추가**

```tsx
const [tab, setTab] = useState<'basic' | 'memory' | 'world'>('basic')
```
패널 헤더 아래에 탭 바:
```tsx
<div className="sp-tabs">
  {([['basic','기본'],['memory','기억'],['world','세계관']] as const).map(([k,l]) => (
    <button key={k} className={`sp-tab ${tab === k ? 'on' : ''}`} onClick={() => setTab(k)}>{l}</button>
  ))}
</div>
```

- [ ] **Step 2: 기존 섹션을 탭별로 묶기** (각 섹션의 내부 JSX는 그대로 유지, 바깥을 `{tab === 'X' && ( ... )}`로 감싸기)

- **basic**: 분기 설명(branches>1), 대화 제목, 대화 참여자(`onShowCharCard`), 내 역할(페르소나 아코디언), 🎨 스타일(기존 style 아코디언).
- **memory**: 📌 기억·상태(핵심 메모리·타임라인), 🧠 장기 메모리, 🔖 북마크.
- **world**: 시나리오 배경, 🔖 AI 자동 챕터, 🗺 스토리 설계도, 📖 로어북.

> Plan 01에서 이미 제거된 "화면 테마 설정"·"대화방 배경 이미지" 섹션은 존재하지 않음(존재 시 제거).

- [ ] **Step 3: 참여자·페르소나 기본 접힘 유지**

페르소나(`panelOpen.persona`)·스타일·기억 등 아코디언 초기값은 접힘. (현재 `panelOpen` 기본값 점검: `memory: true`로 되어 있으면 `false`로) — 기본 진입 시 모두 접힘.

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

- [ ] **Step 5: 커밋**

```bash
git add "apps/web/app/(main)/conversations/[id]/_components/SidePanel.tsx"
git commit -m "Feat: 대화 설정 패널 3탭(기본/기억/세계관) 재편"
```

---

### Task 5: 캐릭터 카드 모달 — 연필/닫기 조정

**Files:**
- Modify: `apps/web/components/ui/CharacterCardModal.tsx`

- [ ] **Step 1: 수정 버튼을 이름 옆 연필로, 하단은 닫기만**

상단 ✕ 제거. 이름 옆에 ✏ 버튼(프리셋 아니면) → `router.push('/characters/:id/edit')`. 하단 액션을 닫기 버튼 하나만(우측).

```tsx
<div className="nmrow">
  <span className="cc-nm">{character.name}</span>
  {!character.isPreset && <button className="cc-edit" onClick={() => router.push(`/characters/${character.id}/edit`)} aria-label="수정">✏</button>}
</div>
...
<div className="cc-acts"><button className="btn ghost" onClick={onClose}>닫기</button></div>
```

- [ ] **Step 2: 빌드 검증·커밋**

Run: `cd apps/web && npx next build`
```bash
git add apps/web/components/ui/CharacterCardModal.tsx
git commit -m "Feat: 캐릭터 카드 모달 — 수정 연필(이름 옆)·닫기만"
```

---

### Task 6: 채팅방 CSS + 유실 방지 전수 확인

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가**

```css
.cr-hb{ width:32px; height:32px; border-radius:9px; display:grid; place-items:center; font-size:15px; color:var(--ink-2); background:var(--pane); border:1px solid var(--chrome-border); cursor:pointer; }
.cr-hb.on{ background:#8b5cf622; border-color:var(--accent); color:#c5b6f5; }
.cr-sub{ font-size:9.5px; color:var(--ink-soft); margin-top:3px; }
.cr-statusbar{ display:flex; align-items:center; gap:6px; width:calc(100% - 8px); margin:8px 4px 0; padding:7px 12px; background:var(--pane); border:1px solid var(--chrome-border); border-radius:9px; cursor:pointer; }
.cr-statusbar .tx{ font-size:9.5px; color:var(--ink-soft); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; flex:1; text-align:left; }
.cr-hmenu{ position:absolute; top:48px; right:11px; z-index:10; width:148px; background:var(--chrome-face); border:1px solid var(--chrome-border); border-radius:11px; overflow:hidden; box-shadow:0 8px 24px rgba(0,0,0,.4); }
.cr-hmenu button{ display:flex; align-items:center; gap:9px; width:100%; padding:10px 12px; font-size:11.5px; color:var(--ink); background:none; border:none; border-bottom:1px solid var(--hairline); cursor:pointer; text-align:left; }
.cr-hmenu button:last-child{ border-bottom:none; }
.cr-hmenu .ic{ width:16px; text-align:center; font-size:13px; }
.cr-user-e{ align-self:flex-end; max-width:84%; margin-left:auto; border-right:3px solid var(--accent); padding:3px 11px 3px 0; text-align:right; color:#b9b3cc; font-style:italic; line-height:1.6; }
.sp-tabs{ display:flex; gap:5px; padding:9px 11px; border-bottom:1px solid var(--chrome-border); }
.sp-tab{ flex:1; text-align:center; font-size:11px; font-weight:700; padding:7px 0; border-radius:8px; color:var(--ink-soft); background:none; border:none; cursor:pointer; }
.sp-tab.on{ background:#8b5cf622; color:#c5b6f5; border:1px solid var(--accent); }
.cc-edit{ width:24px; height:24px; border-radius:7px; background:#8b5cf61f; border:1px solid #8b5cf640; color:#c5b6f5; font-size:11px; cursor:pointer; }
```

- [ ] **Step 2: 유실 방지 체크리스트 수동 전수 확인**

`npm run dev` → 채팅방에서 다음 전부 동작 확인(스펙 "유실 방지 체크리스트"):
1. 메시지 탭 → 재생성/편집/분기/삭제/북마크/음성재생
2. 재회 인사 배너(24h+ 경과 시) 인사받기/무시
3. `/` 슬래시 커맨드 메뉴, `!상태창`/`!스탯`/`!도움말` 즉시 응답
4. 이전 메시지 더 보기, "새 답변 ↓", `?msg=` 점프 하이라이트, 타이핑 아바타, 주사위 굴림 오버레이
5. sendError 재시도
6. ＋메뉴(음성입력/스탯판정/관전/줄거리요약/글자크기), 추천답변 칩
7. ☰ 메뉴 → 스탯/인벤/줄거리/통화 팝오버, ⚙ → 3탭 패널, 분기 칩, E안 유저 메시지

- [ ] **Step 3: 커밋**

```bash
git add apps/web/app/globals.css
git commit -m "Style: 채팅방 헤더/상태줄/☰메뉴/유저메시지(E)/설정탭 스타일"
```

---

## Self-Review
- 스펙 채팅방·설정 패널(3탭)·캐릭터 모달 + 유실 방지 체크리스트 → Task 1~6 커버, Task 6 Step 2가 전수 확인 게이트.
- 기능 로직은 보존(마크업/스타일/그룹핑만 변경). 플레이스홀더 없음.
- 타입 일관성: `showHeaderMenu` state 신규, 기존 `showStats/showInventory/showRecap/showVoiceCall/showPanel` 재사용.
