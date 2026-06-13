# 레이아웃 재설계 02 — 전역 셸(헤더·계정 메뉴·독) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 모든 페이지 공통 헤더(StoryFit→홈 + 🙂 계정 메뉴)와 5탭 독(설정 자리를 🤖 챗봇으로 교체)을 구성하고, 채팅방에서는 독을 숨긴다.

**Architecture:** `(main)/layout.tsx`의 shell-title을 브랜드 헤더로 교체하고 우측에 계정 메뉴(🔧 관리자 admin만 / ⚙️ 설정 / ⏻ 로그아웃)를 단다. `Dock`의 설정 탭을 챗봇(`/assistant`)으로 바꾸고, 채팅방 경로일 때 `<Dock>`을 렌더하지 않는다.

**Tech Stack:** Next.js 14 App Router, React 클라이언트 컴포넌트.

**선행:** Plan 01(다크 단일화) 완료.

**검증 방식:** `npx tsc --noEmit`, `npx next build`, 수동 확인. (apps/web 기준)

---

## File Structure
- 생성: `apps/web/components/shell/AccountMenu.tsx` — 🙂 + 드롭다운(관리자/설정/로그아웃)
- 수정: `apps/web/components/shell/Dock.tsx` — 설정 → 챗봇(/assistant)
- 수정: `apps/web/app/(main)/layout.tsx` — 브랜드 헤더 + AccountMenu, 채팅방 독 숨김
- 수정: `apps/web/app/globals.css` — 브랜드/계정 메뉴 스타일

---

### Task 1: AccountMenu 컴포넌트

**Files:**
- Create: `apps/web/components/shell/AccountMenu.tsx`

- [ ] **Step 1: 컴포넌트 작성**

```tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { getIsAdmin, apiLogout } from '@/lib/authClient'

export default function AccountMenu() {
  const router = useRouter()
  const [open, setOpen] = useState(false)
  const isAdmin = getIsAdmin()

  const logout = async () => { await apiLogout(); router.replace('/login') }

  return (
    <div style={{ position: 'relative' }}>
      <button
        className="acct-btn"
        aria-label="계정 메뉴"
        onClick={() => setOpen(o => !o)}
      >🙂</button>
      {open && (
        <>
          <div style={{ position: 'fixed', inset: 0, zIndex: 40 }} onClick={() => setOpen(false)} />
          <div className="acct-menu">
            {isAdmin && (
              <button className="acct-item" onClick={() => { setOpen(false); router.push('/admin') }}>
                <span className="ic">🔧</span>관리자
              </button>
            )}
            <button className="acct-item" onClick={() => { setOpen(false); router.push('/settings') }}>
              <span className="ic">⚙️</span>설정
            </button>
            <button className="acct-item danger" onClick={() => { setOpen(false); logout() }}>
              <span className="ic">⏻</span>로그아웃
            </button>
          </div>
        </>
      )}
    </div>
  )
}
```

- [ ] **Step 2: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: AccountMenu 관련 에러 없음.

- [ ] **Step 3: 커밋**

```bash
git add apps/web/components/shell/AccountMenu.tsx
git commit -m "Feat: 계정 메뉴 컴포넌트(관리자/설정/로그아웃) 추가 (셸 1/4)"
```

---

### Task 2: Dock 설정 → 챗봇 교체

**Files:**
- Modify: `apps/web/components/shell/Dock.tsx:17-25`

- [ ] **Step 1: 설정 탭을 챗봇으로 교체**

`Dock.tsx`의 `TABS` 마지막 항목(설정)을 아래로 교체:

```tsx
  {
    href: '/assistant', icon: '🤖', label: '챗봇',
    isActive: (p: string) => p.startsWith('/assistant'),
  },
```

그리고 채팅 탭의 `isActive`에서 `/assistant`를 제거(챗봇 탭으로 이관):

```tsx
  {
    href: '/chatlist', icon: '💬', label: '채팅',
    isActive: (p: string) => p === '/chatlist' || p.startsWith('/conversations'),
  },
```

- [ ] **Step 2: 빌드 검증**

Run: `cd apps/web && npx next build`
Expected: 성공.

- [ ] **Step 3: 커밋**

```bash
git add apps/web/components/shell/Dock.tsx
git commit -m "Feat: 독 설정 탭을 챗봇(/assistant)으로 교체 (셸 2/4)"
```

---

### Task 3: (main) 레이아웃 — 브랜드 헤더 + 계정 메뉴 + 채팅방 독 숨김

**Files:**
- Modify: `apps/web/app/(main)/layout.tsx`

- [ ] **Step 1: Shell 헤더 교체 및 독 조건부 렌더**

`Shell` 컴포넌트의 `shell-title` div를 아래 헤더로 교체하고, `<Dock />`을 `{!isChatPage && <Dock />}`로 변경. `AccountMenu` import 추가.

```tsx
import AccountMenu from '@/components/shell/AccountMenu'
```

헤더(기존 `shell-title` 블록 대체):
```tsx
        <div className="app-header">
          <button className="brand" onClick={() => router.push('/')} aria-label="홈으로">
            <svg viewBox="0 0 16 16" width="15" height="15" shapeRendering="crispEdges">
              <rect x="2" y="2" width="12" height="12" fill="#ff8fcf"/>
              <rect x="3" y="3" width="10" height="10" fill="#ffe07a"/>
              <rect x="6" y="5" width="1" height="1" fill="#1a1438"/>
              <rect x="9" y="5" width="1" height="1" fill="#1a1438"/>
              <rect x="6" y="8" width="4" height="1" fill="#1a1438"/>
            </svg>
            <span>StoryFit</span>
          </button>
          <AccountMenu />
        </div>
```

독 렌더부:
```tsx
        {!isChatPage && <Dock />}
```

- [ ] **Step 2: 사용 안 하는 SCREEN_LABELS/label 정리**

헤더가 항상 "StoryFit"이므로 `SCREEN_LABELS`/`label` 계산이 더 이상 헤더에 쓰이지 않으면 제거(다른 곳에서 안 쓰면). 사용처 없으면 관련 변수 삭제.

- [ ] **Step 3: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

- [ ] **Step 4: 커밋**

```bash
git add "apps/web/app/(main)/layout.tsx"
git commit -m "Feat: 전역 브랜드 헤더+계정 메뉴, 채팅방 독 숨김 (셸 3/4)"
```

---

### Task 4: 헤더·계정 메뉴 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가**

globals.css 끝에 추가:
```css
.app-header{ display:flex; align-items:center; justify-content:space-between; padding:11px 14px; border-bottom:1px solid var(--hairline); }
.app-header .brand{ display:flex; align-items:center; gap:6px; background:none; border:none; color:var(--ink); font-size:13px; font-weight:800; cursor:pointer; padding:0; }
.acct-btn{ width:28px; height:28px; border-radius:50%; background:var(--pane); border:1px solid var(--chrome-border); font-size:14px; display:grid; place-items:center; cursor:pointer; }
.acct-menu{ position:absolute; top:36px; right:0; z-index:41; width:140px; background:var(--chrome-face); border:1px solid var(--chrome-border); border-radius:11px; overflow:hidden; box-shadow:0 8px 24px rgba(0,0,0,.4); }
.acct-item{ display:flex; align-items:center; gap:9px; width:100%; padding:10px 12px; font-size:12px; color:var(--ink); background:none; border:none; border-bottom:1px solid var(--hairline); cursor:pointer; }
.acct-item:last-child{ border-bottom:none; }
.acct-item.danger{ color:#ff7a90; }
.acct-item .ic{ width:15px; text-align:center; }
```

- [ ] **Step 2: 빌드·수동 확인**

Run: `cd apps/web && npx next build`
이후 `npm run dev`: 모든 페이지 상단에 StoryFit(클릭 시 홈) + 🙂(클릭 시 관리자[admin만]/설정/로그아웃) 노출. 채팅방에서 하단 독 사라짐. 독 5번째 탭이 🤖 챗봇.

- [ ] **Step 3: 커밋**

```bash
git add apps/web/app/globals.css
git commit -m "Style: 브랜드 헤더·계정 메뉴 스타일 (셸 4/4)"
```

---

## Self-Review
- 스펙 "전역 헤더 / 🙂 계정 메뉴 / 독 5탭(챗봇) / 채팅방 독 숨김" → Task 1~4 커버.
- 유실 방지: 관리자·설정·로그아웃 진입점이 계정 메뉴로 보존, 어시스턴트 진입점이 챗봇 탭으로 복구.
- 플레이스홀더 없음.
