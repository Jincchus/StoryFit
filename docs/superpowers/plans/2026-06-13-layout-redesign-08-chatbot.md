# 레이아웃 재설계 08 — 챗봇(어시스턴트) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 챗봇(어시스턴트) 목록과 대화방을 새 다크 디자인으로 정비한다(스토리 요소 없는 자유 대화).

**Architecture:** `assistant/page.tsx`(목록)와 `assistant/[id]/page.tsx`(대화방)의 데이터·생성·삭제 로직은 유지하고 마크업/스타일만 교체한다. 목록은 봇 아바타 행, 대화방은 유저=우측 말풍선·AI=좌측 말풍선.

**Tech Stack:** Next.js 14, React.

**선행:** Plan 01·02(챗봇 독 탭).

**검증:** `npx tsc --noEmit`, `npx next build`, 수동 확인. 참고 목업: `content/chatbot-list.html`, `chatbot-room.html`.

---

## File Structure
- 수정: `apps/web/app/(main)/assistant/page.tsx` — 목록 행 마크업
- 수정: `apps/web/app/(main)/assistant/[id]/page.tsx` — 대화방 말풍선/헤더 (Task 0에서 현 구조 확인)
- 수정: `apps/web/app/globals.css` — 챗봇 목록/말풍선 스타일

---

### Task 0: 어시스턴트 대화방 현 구조 확인

- [ ] **Step 1: 파일 확인**

Run: `cd apps/web && sed -n '1,80p' "app/(main)/assistant/[id]/page.tsx"` (또는 Read)
헤더·메시지 렌더·컴포저 구조와 재사용 컴포넌트(MessageList 사용 여부)를 파악해 Task 2 마크업을 실제 구조에 맞춘다.

---

### Task 1: 챗봇 목록 마크업

**Files:**
- Modify: `apps/web/app/(main)/assistant/page.tsx`

- [ ] **Step 1: 헤더·행 교체**

헤더: "챗봇 / 롤플레이 없이 AI와 자유롭게 대화" + ✦ 새 대화(`handleNew` 유지). 행:

```tsx
{conversations.map(conv => (
  <button key={conv.id} className="bot-row" onClick={() => router.push(`/assistant/${conv.id}`)}>
    <div className="bot-av">🤖</div>
    <div className="bot-mid">
      <div className="bot-t">{conv.title}</div>
      <div className="bot-q">{previewText(conv.messages[0]?.content ?? '')}</div>
    </div>
    <div className="bot-right">
      <span className="bot-when">{new Date(conv.updatedAt).toLocaleDateString('ko-KR', { month:'short', day:'numeric' })}</span>
      <span className="bot-del" onClick={e => { e.stopPropagation(); setConfirmDeleteId(conv.id) }}>✕</span>
    </div>
  </button>
))}
```
빈 상태/로딩/ConfirmDelete는 기존 유지.

- [ ] **Step 2: 빌드 검증**

Run: `cd apps/web && npx next build`
Expected: 성공.

---

### Task 2: 챗봇 대화방 말풍선

**Files:**
- Modify: `apps/web/app/(main)/assistant/[id]/page.tsx`

- [ ] **Step 1: 메시지 렌더 교체** (Task 0 구조 기준)

유저=우측 말풍선(`bot-me`), AI=좌측 봇 아바타+말풍선(`bot-ai-row`/`bot-ai`). 헤더는 ← 뒤로 + 🤖 + 제목(✏ 수정은 기존 rename 로직 있으면 연결, 없으면 생략) + ✦ 새 대화. 컴포저(입력+전송)는 기존 유지.

```tsx
// 메시지 루프
m.role === 'user'
  ? <div className="bot-me">{m.content}</div>
  : <div className="bot-ai-row"><span className="bot-ai-av">🤖</span><div className="bot-ai">{m.content}</div></div>
```

- [ ] **Step 2: 타입·빌드 검증**

Run: `cd apps/web && npx tsc --noEmit && npx next build`
Expected: 에러 0, 성공.

---

### Task 3: 챗봇 CSS

**Files:**
- Modify: `apps/web/app/globals.css`

- [ ] **Step 1: 스타일 추가**

```css
.bot-row{ display:flex; gap:10px; padding:10px; border-radius:12px; background:var(--paper); border:1px solid var(--hairline); align-items:center; width:100%; text-align:left; cursor:pointer; color:var(--ink); }
.bot-av{ width:42px; height:42px; border-radius:11px; flex-shrink:0; background:linear-gradient(135deg,#2a2238,#1c1c28); border:1px solid #3a335a; display:grid; place-items:center; font-size:20px; }
.bot-mid{ flex:1; min-width:0; }
.bot-t{ font-size:12px; font-weight:800; color:var(--ink); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.bot-q{ font-size:10px; color:var(--ink-soft); margin-top:3px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.bot-right{ display:flex; flex-direction:column; align-items:flex-end; gap:6px; flex-shrink:0; }
.bot-when{ font-size:9px; color:var(--ink-faint); }
.bot-del{ font-size:11px; color:var(--ink-faint); cursor:pointer; }

.bot-me{ align-self:flex-end; max-width:78%; background:linear-gradient(135deg,#6d28d9,#8b5cf6); color:#fff; font-size:11.5px; line-height:1.55; padding:10px 13px; border-radius:14px 14px 4px 14px; }
.bot-ai-row{ display:flex; gap:8px; align-items:flex-start; }
.bot-ai-av{ width:26px; height:26px; border-radius:8px; background:linear-gradient(135deg,#2a2238,#1c1c28); border:1px solid #3a335a; display:grid; place-items:center; font-size:13px; flex-shrink:0; }
.bot-ai{ background:var(--paper); border:1px solid var(--hairline); border-radius:4px 14px 14px 14px; padding:10px 13px; font-size:11.5px; line-height:1.65; color:var(--ink-2); max-width:78%; }
```

- [ ] **Step 2: 수동 확인·커밋**

`npm run dev` → 독 🤖 챗봇 탭: 목록(봇 행·새 대화·삭제), 대화방(유저 우측/AI 좌측 말풍선·전송). 스토리 요소 없음.

```bash
git add "apps/web/app/(main)/assistant/page.tsx" "apps/web/app/(main)/assistant/[id]/page.tsx" apps/web/app/globals.css
git commit -m "Feat: 챗봇 목록·대화방 새 디자인"
```

---

## Self-Review
- 스펙 챗봇(목록·대화방) → Task 1~3. 대화방 실제 구조는 Task 0 확인 후 마크업 확정.
- 유실 방지: 새 대화 생성·삭제·기록 로딩 로직 유지.
