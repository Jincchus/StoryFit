# 레이아웃 재설계 04 — 채팅목록 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 채팅목록을 새 다크 디자인 리스트로 재정비하되, 검색·본문검색·필터·정렬·핀·서재·삭제·선택모드·자동생성 진입을 전부 보존한다.

**Architecture:** `chatlist/page.tsx`의 기능 로직(상태·핸들러·검색·필터)은 유지하고 행 마크업과 헤더만 새 디자인으로 교체한다. 제목은 `채팅방명(캐릭터.페르소나)` 형식, 우측에 센터 배지.

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01·02.

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인. 참고 목업: `content/chatlist-v2.html`.

---

## File Structure
- 생성: `apps/web/lib/convDisplay.ts` — 공용 제목/센터/모드 헬퍼 (홈·서재와 공유)
- 수정: `apps/web/app/(main)/chatlist/page.tsx` — 헤더·행 마크업 교체
- 수정: `apps/web/app/globals.css` — 채팅목록 행 스타일

---

### Task 1: 공용 표시 헬퍼 추출

**Files:**
- Create: `apps/web/lib/convDisplay.ts`

- [ ] **Step 1: 헬퍼 작성**

```ts
export function getCenter(sourceUrl?: string): 'WHIF' | 'ZETA' | 'MELTING' | null {
  if (!sourceUrl) return null
  if (sourceUrl.includes('zeta-ai.io')) return 'ZETA'
  if (sourceUrl.includes('melting.chat')) return 'MELTING'
  if (sourceUrl.includes('whif.')) return 'WHIF'
  return null
}

export const CENTER_COLOR: Record<string, string> = { WHIF: '#8b5cf6', ZETA: '#7c5cff', MELTING: '#ff2e93' }
export const MODE_LABEL: Record<string, string> = { story: '스토리', multiStory: '멀티' }

export function convTitleParts(c: {
  title: string
  characters: { character: { name: string } }[]
  personaCharacter?: { name: string } | null
}): { room: string; chars: string; persona: string } {
  const names = c.characters.map(x => x.character.name)
  const chars = names.length <= 1 ? (names[0] ?? '') : `${names[0]},${names[1]}${names.length > 2 ? '…' : ''}`
  return { room: c.title, chars, persona: c.personaCharacter?.name ?? '' }
}
```

- [ ] **Step 2: 타입 검증**

Run: `cd apps/web && npx tsc --noEmit`
Expected: 에러 없음.

> 참고: Plan 03 홈의 인라인 `getCenter`/`convTitle`은 이후 이 모듈 import로 교체 권장(중복 제거).

---

### Task 2: 채팅목록 헤더·행 마크업 교체

**Files:**
- Modify: `apps/web/app/(main)/chatlist/page.tsx`

- [ ] **Step 1: import 추가**

```tsx
import { getCenter, CENTER_COLOR, MODE_LABEL, convTitleParts } from '@/lib/convDisplay'
```
기존 로컬 `getSource`/`MODE_LABEL`/`SOURCE_BADGE_COLOR`는 유지해도 무방하나, 행에서는 `getCenter`/`convTitleParts` 사용.

- [ ] **Step 2: 행(`filtered.map`) 마크업 교체**

기존 `.row.swipe-content` 내부 마크업을 새 구조로 교체. 검색/필터/선택모드/스와이프 래퍼(`swipe-wrap`/`swipe-actions`)·핸들러는 그대로 유지하고, `.row` 내부만 교체:

```tsx
<div className="cl-row" ...기존 onClick/onTouch 유지...>
  {selecting && (<div className="cl-check">{isChecked ? '✓' : ''}</div>)}
  <div className="cl-th">
    {char?.avatarUrl ? <img src={char.avatarUrl} alt="" /> : <PixelAvatar kind={char?.kind as any} size={44} />}
  </div>
  <div className="cl-mid">
    <div className="cl-t">
      {conv.isPinned && <span className="cl-pin">📌</span>}
      {(() => { const t = convTitleParts(conv); return <>{t.room}<span className="cl-pp">({t.chars}{t.persona ? `.${t.persona}` : ''})</span></> })()}
    </div>
    <div className="cl-q">{previewText(conv.messages[0]?.content ?? '')}</div>
    <div className="cl-sub">
      {conv.isAutoCreated && <span className="cl-setup">설정 필요</span>}
      <span className="md">{MODE_LABEL[conv.mode] ?? conv.mode}</span>
      {conv.autoChapterEnabled && (conv.chapter ?? 1) > 1 && <> · {conv.chapter}장</>}
      {' · '}{timeAgoShort(conv.updatedAt)}
    </div>
  </div>
  <div className="cl-right">
    {getCenter(conv.sourceUrl) && <span className="cl-center" style={{ background: CENTER_COLOR[getCenter(conv.sourceUrl)!] }}>{getCenter(conv.sourceUrl)}</span>}
  </div>
</div>
```

`timeAgoShort`는 기존 `when`(toLocaleString) 또는 간단 포맷 사용 — 기존 `when` 변수를 `cl-sub`에 그대로 써도 됨.

- [ ] **Step 3: 헤더 텍스트 정리**

상단 "최근 대화 / N개의 진행 중인 롤플레이"는 유지 가능. 본문 검색 결과 패널·필터 칩·정렬 버튼은 **그대로 유지**(마크업 변경 없이 다크 토큰으로 자동 적용). 인라인 액션 버튼(📌/📚/✕)은 스와이프로 대체되므로 `row-inline-actions`는 현행 유지.

- [ ] **Step 4: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 3: 채팅목록 행 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가**

```css
.cl-row{ display:flex; gap:10px; padding:9px 10px; border-radius:12px; background:var(--paper); border:1px solid var(--hairline); align-items:center; }
.cl-row .cl-th{ width:46px; height:46px; border-radius:10px; overflow:hidden; flex-shrink:0; }
.cl-row .cl-th img{ width:100%; height:100%; object-fit:cover; }
.cl-check{ width:20px; flex-shrink:0; display:grid; place-items:center; color:var(--accent); font-weight:800; }
.cl-mid{ flex:1; min-width:0; }
.cl-t{ font-size:12px; font-weight:800; color:var(--ink); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.cl-pin{ color:var(--hot-pink); margin-right:3px; font-size:10px; }
.cl-pp{ color:var(--accent); font-weight:600; }
.cl-q{ font-size:10px; color:var(--ink-soft); margin-top:3px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.cl-sub{ font-size:9px; color:var(--ink-faint); margin-top:4px; }
.cl-sub .md{ color:var(--accent); font-weight:700; }
.cl-setup{ font-size:8px; font-weight:800; color:#fff; background:#4fa8e8; padding:1px 6px; border-radius:20px; margin-right:5px; }
.cl-right{ flex-shrink:0; align-self:flex-start; }
.cl-center{ font-size:8px; font-weight:800; color:#fff; padding:2px 6px; border-radius:20px; }
```
핀 고정 행 좌측 강조는 기존 `borderLeft` 인라인 스타일 유지 또는 `.cl-row`에 조건부 클래스로.

- [ ] **Step 2: 수동 확인**

`npm run dev` → /chatlist: 행이 새 카드 스타일, 제목 `방명(캐릭터.페르소나)`, 우측 센터 배지, 검색어 입력 시 본문 검색 패널·`?msg=` 점프 동작, 모드/소스 필터·정렬·핀·스와이프(📌/📚/🗑)·선택모드·일괄삭제 정상.

- [ ] **Step 3: 커밋**

```bash
git add apps/web/lib/convDisplay.ts "apps/web/app/(main)/chatlist/page.tsx" apps/web/app/globals.css
git commit -m "Feat: 채팅목록 새 디자인 리스트(기능 보존)"
```

---

## Self-Review
- 유실 방지: 검색·본문검색·필터(모드/소스)·정렬·핀·서재(archive)·삭제·선택모드·자동생성 진입 전부 로직 유지, 마크업만 교체.
- 플레이스홀더 없음. `timeAgoShort`는 기존 `when` 재사용 명시.
