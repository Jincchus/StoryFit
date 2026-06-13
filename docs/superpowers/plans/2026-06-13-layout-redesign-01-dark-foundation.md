# 레이아웃 재설계 01 — 다크 단일화 + 테마/배경 제거 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 다중 테마 시스템과 대화방 배경 이미지 기능을 제거하고, 다크 토큰을 `:root` 기본으로 승격해 앱을 다크 단일 테마로 만든다.

**Architecture:** 현재 `:root`는 라이트(IG) 토큰, 다크는 `body[data-art="dark"]` 스코프에 정의돼 있다. 다크 값을 `:root`로 옮기고 `data-art` 의존을 전부 제거한다. 테마 로더(`lib/theme.ts`, `ThemeProvider`, pre-paint 스크립트, `/themes/*.css`)와 테마/배경 UI(설정 테마 탭, 설정 패널 테마·배경 입력, 채팅방 `customBg`)를 삭제한다.

**Tech Stack:** Next.js 14 App Router, CSS custom properties, Prisma, vitest.

**검증 방식:** 컴포넌트 단위 테스트가 없으므로 각 변경은 `npx tsc --noEmit`(타입), `npx vitest run`(기존 회귀), `npx next build`(빌드), 그리고 명시된 수동 확인으로 검증한다. 모든 명령은 `apps/web`에서 실행한다.

---

## File Structure

- 수정: `apps/web/app/globals.css` — `:root`를 다크 토큰으로 교체, `body[data-art=...]`/`body[data-palette=...]` 의존 제거
- 수정: `apps/web/app/layout.tsx` — pre-paint 테마 스크립트·`data-art`·`ThemeProvider` 제거
- 삭제: `apps/web/components/ThemeProvider.tsx`
- 삭제: `apps/web/lib/theme.ts`
- 삭제: `apps/web/public/themes/` (전체)
- 수정: `apps/web/app/(main)/settings/page.tsx` — 테마 탭 제거
- 삭제: `apps/web/app/(main)/settings/_components/ThemeTab.tsx`
- 수정: `apps/web/app/api/user/settings/route.ts` — `theme` 선택/저장 제거
- 수정: `apps/web/prisma/schema.prisma` — `User.theme` 컬럼 제거(선택, db push 동반)
- 수정: `apps/web/app/(main)/conversations/[id]/_components/SidePanel.tsx` — 테마 select·배경 이미지 섹션 + 관련 props 제거
- 수정: `apps/web/app/(main)/conversations/[id]/page.tsx` — `customBg`/`currentTheme` state·렌더·props 제거
- 수정: `apps/web/app/(main)/conversations/[id]/_lib/chatShared.ts` (필요 시 타입에서 theme 관련 제거)

---

### Task 1: globals.css 다크 토큰을 :root로 승격

**Files:**
- Modify: `apps/web/app/globals.css:4-95`, `apps/web/app/globals.css:1262-1264`

- [ ] **Step 1: `:root` 블록(4-44행)을 다크 토큰으로 교체**

`apps/web/app/globals.css`의 기존 `:root{ ... }`(라이트 IG, 4-44행)를 아래로 교체:

```css
:root{
  --bg:           #0d0d0f;
  --bg-2:         #1a1a20;
  --paper:        #121215;

  --line:         #2c2c34;
  --hairline:     #222229;
  --hairline-2:   #1b1b21;

  --ink:          #f4f4f6;
  --ink-2:        #d8d8de;
  --ink-soft:     #9a9aa6;
  --ink-faint:    #62626e;

  --accent:       #8b5cf6;
  --accent-2:     #7c3aed;
  --red:          #ed4956;
  --green:        #00c853;

  --ig-gradient:  linear-gradient(45deg, #f09433 0%, #e6683c 20%, #dc2743 45%, #cc2366 70%, #bc1888 100%);
  --bubble-other: #232329;
  --bubble-you:   linear-gradient(195deg, #7b2ff7 0%, #c850c0 50%, #ff6f9c 100%);
  --wall-grad:    linear-gradient(180deg, #0d0d0f, #111118);

  --font-body:   "Pretendard", -apple-system, BlinkMacSystemFont, "Helvetica Neue",
                  "Apple SD Gothic Neo", "Segoe UI", system-ui, sans-serif;
  --font-title:  "Pretendard", -apple-system, BlinkMacSystemFont, "Helvetica Neue",
                  "Apple SD Gothic Neo", system-ui, sans-serif;
  --font-mono:   ui-monospace, "SF Mono", Menlo, monospace;
  --font-script: "Brush Script MT", "Lucida Handwriting", cursive;

  --radius-sm: 6px;
  --radius:    8px;
  --radius-lg: 12px;

  --hot-pink:      #ff2e93;
  --lavender:      rgba(139,92,246,.35);
  --chrome-face:   #1a1a20;
  --chrome-border: #2c2c34;
  --pane:          #1a1a20;
}
```

- [ ] **Step 2: 라이트/팔레트 잔재 + 다크 토큰 중복 블록 삭제**

기존 46-81행(아래 내용)을 통째로 삭제:

```css
body[data-art="modern"]{ }
body[data-art="retro"]{ }

body[data-palette="teal"]    { --wall-grad: linear-gradient(180deg, #fafafa, #f0f0f0) }
body[data-palette="purple"]  { --wall-grad: linear-gradient(180deg, #fff0fa, #f0e6ff) }
body[data-palette="maroon"]  { --wall-grad: linear-gradient(180deg, #fff5e8, #ffe8e8) }
body[data-palette="olive"]   { --wall-grad: linear-gradient(180deg, #f0f4ff, #fdf5ff) }
:root{ --wall-grad: linear-gradient(180deg, #fafafa, #f0f0f0) }

/* ─── StoryFit Dark (기본 테마) … ─ */
body[data-art="dark"]{
  --bg: #0d0d0f; … --pane: #1a1a20;
}
```
(주석 `/* ─── StoryFit Dark … */`와 그 아래 토큰 블록까지 포함해 삭제. 토큰은 Step 1에서 `:root`로 이미 이동함.)

- [ ] **Step 3: 남은 `body[data-art="dark"]` 셀렉터를 `body`로 일괄 치환**

globals.css 전체에서 `body[data-art="dark"]` → `body` 로 모두 치환(82-95행 override 규칙, 1262-1264행 dice 색상 포함). 다크가 유일 테마이므로 무조건 적용되어야 한다.

- [ ] **Step 4: 빌드 검증**

Run: `cd apps/web && npx next build`
Expected: Compiled successfully (CSS 파싱 에러 없음)

- [ ] **Step 5: 커밋**

```bash
git add apps/web/app/globals.css
git commit -m "Refactor: 다크 토큰을 :root 기본으로 승격 (다크 단일화 1/N)"
```

---

### Task 2: 테마 런타임 로더 제거

**Files:**
- Modify: `apps/web/app/layout.tsx`
- Delete: `apps/web/components/ThemeProvider.tsx`, `apps/web/lib/theme.ts`, `apps/web/public/themes/`

- [ ] **Step 1: `layout.tsx`에서 pre-paint 스크립트·data-art·ThemeProvider 제거**

`apps/web/app/layout.tsx` 전체를 아래로 교체:

```tsx
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'StoryFit',
  description: '소설형 롤플레이 AI 채팅',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  )
}
```

- [ ] **Step 2: 파일 삭제**

```bash
cd apps/web
rm components/ThemeProvider.tsx
rm lib/theme.ts
rm -rf public/themes
```

- [ ] **Step 3: theme 모듈 잔여 import 확인**

Run: `cd apps/web && grep -rn "lib/theme\|ThemeProvider\|applyTheme\|getSavedTheme\|THEMES\|EXTERNAL_THEMES" app components lib`
Expected: SidePanel.tsx·ThemeTab.tsx·settings/page.tsx 외 참조 없음(이들은 Task 3·5에서 처리). 그 외 파일에서 나오면 함께 정리.

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: 에러는 Task 3·5 대상 파일(ThemeTab, SidePanel, settings/page)에서만 발생. 다른 파일 에러 시 import 정리.

- [ ] **Step 5: 커밋**

```bash
git add -A
git commit -m "Refactor: 테마 런타임 로더·테마 CSS 제거 (다크 단일화 2/N)"
```

---

### Task 3: 설정 페이지 테마 탭 제거

**Files:**
- Modify: `apps/web/app/(main)/settings/page.tsx`
- Delete: `apps/web/app/(main)/settings/_components/ThemeTab.tsx`

- [ ] **Step 1: ThemeTab import·탭 정의·렌더 제거**

`settings/page.tsx`에서 다음을 제거:
- `import ThemeTab from './_components/ThemeTab'`
- `Tab` 타입에서 `'theme'`
- `TABS` 배열에서 `{ id: 'theme', label: '테마' }`
- 렌더부 `{tab === 'theme' && <ThemeTab />}`

결과 `TABS`:
```tsx
const TABS: { id: Tab; label: string }[] = [
  { id: 'profile', label: '프로필·프롬프트' },
  { id: 'params', label: '파라미터' },
  { id: 'security', label: '보안' },
  { id: 'stats', label: '통계' },
  { id: 'export', label: '내보내기' },
]
```
그리고 `type Tab = 'profile' | 'params' | 'security' | 'stats' | 'export'`.

- [ ] **Step 2: ThemeTab 삭제**

```bash
cd apps/web && rm "app/(main)/settings/_components/ThemeTab.tsx"
```

- [ ] **Step 3: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: settings/page.tsx·ThemeTab 관련 에러 없음(SidePanel은 Task 5에서).

- [ ] **Step 4: 커밋**

```bash
git add -A
git commit -m "Refactor: 설정 테마 탭 제거 (다크 단일화 3/N)"
```

---

### Task 4: 설정 API의 theme 필드 제거

**Files:**
- Modify: `apps/web/app/api/user/settings/route.ts`
- Modify: `apps/web/prisma/schema.prisma`

- [ ] **Step 1: settings route에서 theme 입출력 제거**

`route.ts:13` select에서 `theme: true,` 줄 삭제. `route.ts:46`의 다음 줄 삭제:
```ts
if (['retro', 'modern', 'modernwhite', 'win95', 'maple', 'qplay', 'crazyarcade', 'block', 'cyworld'].includes(body.theme)) data.theme = body.theme
```

- [ ] **Step 2: Prisma 스키마에서 User.theme 제거**

`prisma/schema.prisma`의 `model User`에서 `theme` 필드 줄을 삭제. (필드명 확인 후 제거)

- [ ] **Step 3: prisma generate**

Run: `cd apps/web && npx prisma generate`
Expected: 성공.

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: theme 참조 에러 없음.

- [ ] **Step 5: 커밋**

```bash
git add apps/web/app/api/user/settings/route.ts apps/web/prisma/schema.prisma
git commit -m "Refactor: 유저 설정에서 theme 필드 제거 (다크 단일화 4/N)"
```

> 배포 시 `docker compose exec web npx prisma db push`로 컬럼 drop 반영 필요(데이터 손실 경고 시 `--accept-data-loss`). 배포 안내에 포함할 것.

---

### Task 5: 설정 패널(SidePanel)에서 테마·배경 제거

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/_components/SidePanel.tsx:148-193`

- [ ] **Step 1: 테마 select 섹션 삭제**

`SidePanel.tsx`의 `{/* 화면 테마 및 배경 설정 */}` 주석 + "화면 테마 설정" `side-section`(148-166행 영역) 삭제. `import { applyTheme, THEMES } from '@/lib/theme'`도 삭제.

- [ ] **Step 2: 배경 이미지 섹션 삭제**

"대화방 배경 이미지 (URL)" `side-section`(168-193행 영역) 삭제.

- [ ] **Step 3: props에서 theme/bg 제거**

SidePanel props에서 `customBg`, `setCustomBg`, `currentTheme`, `setCurrentTheme` 4개 제거(구조분해 및 타입 정의 양쪽).

- [ ] **Step 4: 타입 검증 (호출부 에러 확인)**

Run: `cd apps/web && npx tsc --noEmit`
Expected: `page.tsx`의 `<SidePanel ... customBg=... />` 호출부에서 prop 불일치 에러 → Task 6에서 해결.

---

### Task 6: 채팅방 page.tsx에서 customBg/currentTheme 제거

**Files:**
- Modify: `apps/web/app/(main)/conversations/[id]/page.tsx`

- [ ] **Step 1: state·로직 제거**

`page.tsx`에서 다음 제거:
- `customBg`, `setCustomBg`, `currentTheme`, `setCurrentTheme` useState 및 초기화(localStorage `sf_bg_`, theme 로드).
- `.chatlog`의 `backgroundImage: customBg ? ...` 및 `customBg && (<div ... 오버레이 />)` 블록(827-845행 영역).
- `<SidePanel>`에 넘기던 `customBg`/`setCustomBg`/`currentTheme`/`setCurrentTheme` 4개 prop(1140-1143행 영역).

- [ ] **Step 2: 잔여 customBg 참조 확인**

Run: `cd apps/web && grep -rn "customBg\|currentTheme\|sf_bg_\|sf-theme" "app/(main)/conversations"`
Expected: 참조 없음.

- [ ] **Step 3: 타입·빌드·회귀 검증**

Run: `cd apps/web && npx tsc --noEmit && npx vitest run && npx next build`
Expected: tsc 에러 0, vitest 전부 PASS, build 성공.

- [ ] **Step 4: 수동 확인**

`npm run dev` 후 브라우저에서: (1) 앱 전체가 다크로 렌더되는지, (2) 설정 페이지에 테마 탭 없음, (3) 채팅방 ⚙ 설정 패널에 테마/배경 항목 없음, (4) 라이트로 깜빡임(FOUC) 없는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add -A
git commit -m "Refactor: 채팅방·설정 패널에서 테마/배경 이미지 기능 제거 (다크 단일화 5/N)"
```

---

## Self-Review
- 스펙 "테마 시스템 제거" / "배경 이미지 제거" / "다크 토큰 :root 승격(선행 필수)" → Task 1~6으로 모두 커버.
- 플레이스홀더 없음(각 CSS/코드 변경 구체 명시). prisma `User.theme` 필드명은 실제 스키마 확인 후 제거하도록 명시.
- 타입 일관성: SidePanel에서 제거하는 prop 4개가 Task 6 호출부 제거와 짝을 이룸.

## 후속 플랜 (별도 작성 예정)
1. **(이 플랜) 다크 단일화 + 테마/배경 제거** ← 선행
2. 전역 셸: StoryFit 헤더(→홈) · 🙂 계정 메뉴(관리자/설정/로그아웃) · 독 5탭(설정→🤖 챗봇) · 채팅방 독 숨김
3. 홈 재구성(이어하기/센터/액션/대화중/완결)
4. 채팅목록(본문검색 패널·필터·스와이프·새 카드 행)
5. 탐색(센터별 최근 가로 스와이프)
6. 서재(완결 카드 그리드 + ⋯ 메뉴)
7. 채팅방 + 설정 패널 3탭(유실 방지 체크리스트 항목 전수 보존)
8. 챗봇(목록·대화방)
9. 설정·관리자 다크 CSS 점검(레이아웃 유지)
