# 홈 이어하기 분기 최신 기준 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 홈 이어하기가 대화별 가장 최근 활동 분기를 보여주고 그 분기로 직접 진입하게 한다.

**Architecture:** 홈 전용 신규 엔드포인트 `GET /api/conversations/recent`가 스레드(root+분기)별 최신 노드를 골라 활성 분기 최신 메시지와 함께 반환. 그룹화 로직은 순수 함수로 분리해 Vitest로 검증. 공용 `/api/conversations`·chatlist는 불변.

**Tech Stack:** Next.js 14 API Routes, Prisma, Vitest, TypeScript.

설계: `context/2026-06-28-home-continue-latest-branch-design.md`

## Global Constraints

- 홈 이어하기만 대상. chatlist·공용 `/api/conversations`·기타 화면 불변.
- 스레드키 = `rootConversationId ?? id`. 스레드별 `updatedAt` 최대 노드 선택, updatedAt desc 정렬, 상위 limit.
- 미리보기는 그 노드의 `isSelected: true` 최신 메시지.
- 작업 디렉터리 `apps/web`. 커밋은 apps/web. 배포는 사용자 확인 하에.
- 검증: `npx tsc --noEmit` + `npx vitest run`.

---

### Task 1: 스레드 최신 노드 선택 순수 함수 + 테스트

**Files:**
- Create: `lib/recentThreads.ts`
- Test: `lib/recentThreads.test.ts`

**Interfaces:**
- Produces:
  - `type ThreadNode = { id: string; rootConversationId: string | null; updatedAt: Date | string }`
  - `pickLatestNodeIdsPerThread(nodes: ThreadNode[], limit: number): string[]`

- [ ] **Step 1: 실패 테스트** — `lib/recentThreads.test.ts`

```ts
import { describe, it, expect } from 'vitest'
import { pickLatestNodeIdsPerThread } from './recentThreads'

const n = (id: string, root: string | null, t: string) => ({ id, rootConversationId: root, updatedAt: t })

describe('pickLatestNodeIdsPerThread', () => {
  it('스레드(root+분기)별 updatedAt 최대 노드를 골라 desc 정렬', () => {
    const out = pickLatestNodeIdsPerThread([
      n('a1', null, '2026-06-01T00:00:00Z'),
      n('a2', 'a1', '2026-06-03T00:00:00Z'),   // 스레드 a 최신 = 분기 a2
      n('b1', null, '2026-06-02T00:00:00Z'),   // 스레드 b
    ], 10)
    expect(out).toEqual(['a2', 'b1'])
  })
  it('limit으로 상위 N개만', () => {
    const out = pickLatestNodeIdsPerThread([
      n('a', null, '2026-06-01T00:00:00Z'),
      n('b', null, '2026-06-02T00:00:00Z'),
      n('c', null, '2026-06-03T00:00:00Z'),
    ], 2)
    expect(out).toEqual(['c', 'b'])
  })
})
```

- [ ] **Step 2: 실패 확인** — Run: `npx vitest run lib/recentThreads.test.ts` → FAIL(모듈 없음).

- [ ] **Step 3: 구현** — `lib/recentThreads.ts`

```ts
export type ThreadNode = { id: string; rootConversationId: string | null; updatedAt: Date | string }

// 스레드(rootConversationId ?? id)별로 updatedAt 최대 노드를 골라 updatedAt desc 정렬, 상위 limit개 id 반환.
export function pickLatestNodeIdsPerThread(nodes: ThreadNode[], limit: number): string[] {
  const best = new Map<string, ThreadNode>()
  for (const node of nodes) {
    const key = node.rootConversationId ?? node.id
    const cur = best.get(key)
    if (!cur || new Date(node.updatedAt).getTime() > new Date(cur.updatedAt).getTime()) {
      best.set(key, node)
    }
  }
  return Array.from(best.values())
    .sort((a, b) => new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime())
    .slice(0, limit)
    .map(node => node.id)
}
```

- [ ] **Step 4: 통과 확인** — Run: `npx vitest run lib/recentThreads.test.ts` → PASS.

- [ ] **Step 5: 커밋**

```bash
git add lib/recentThreads.ts lib/recentThreads.test.ts
git commit -m "feat: pickLatestNodeIdsPerThread (스레드별 최신 분기 선택)"
```

---

### Task 2: `/api/conversations/recent` 라우트 + 홈 연결

**Files:**
- Create: `app/api/conversations/recent/route.ts`
- Modify: `app/(main)/page.tsx` (fetch URL)

**Interfaces:**
- Consumes: `pickLatestNodeIdsPerThread`(Task 1).
- Produces: `GET /api/conversations/recent?limit=N` → `Array<{ id, title, mode, updatedAt, characters: [{ character: { id, name, avatarUrl } }], messages: [{ content }] }>` (홈 `RecentConv` 호환), 최신 분기 기준·정렬·제한 적용.

- [ ] **Step 1: 라우트 작성** — `app/api/conversations/recent/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { authenticate } from '@/lib/apiAuth'
import { pickLatestNodeIdsPerThread } from '@/lib/recentThreads'

export async function GET(req: NextRequest) {
  const userId = await authenticate(req)
  if (!userId) return NextResponse.json({ error: '인증이 필요합니다.' }, { status: 401 })

  const { searchParams } = new URL(req.url)
  const limit = Math.min(20, Math.max(1, parseInt(searchParams.get('limit') ?? '3') || 3))

  // 1) 경량: 모든 노드(root+분기)의 스레드 키·시각만
  const nodes = await prisma.conversation.findMany({
    where: { userId, isArchived: false, mode: { not: 'assistant' } },
    select: { id: true, rootConversationId: true, updatedAt: true },
  })
  const ids = pickLatestNodeIdsPerThread(nodes, limit)
  if (ids.length === 0) return NextResponse.json([])

  // 2) 상위 N개만 풀 조회(활성 분기 isSelected 최신 메시지)
  const convs = await prisma.conversation.findMany({
    where: { id: { in: ids } },
    select: {
      id: true, title: true, mode: true, updatedAt: true,
      characters: { include: { character: { select: { id: true, name: true, avatarUrl: true } } } },
      messages: { where: { isSelected: true }, orderBy: { createdAt: 'desc' }, take: 1, select: { content: true } },
    },
  })
  // ids 정렬 순서 유지
  const byId = new Map(convs.map(c => [c.id, c]))
  const ordered = ids.map(id => byId.get(id)).filter(Boolean)
  return NextResponse.json(ordered)
}
```

- [ ] **Step 2: 홈 fetch URL 교체** — `app/(main)/page.tsx`의

```ts
    api.get('/api/conversations')
```
를

```ts
    api.get('/api/conversations/recent?limit=3')
```
로 변경(이후 정렬·`slice(0,3)`는 그대로 둬도 무해 — 엔드포인트가 이미 정렬·제한함).

- [ ] **Step 3: 타입체크 + 테스트**

Run: `npx tsc --noEmit && npx vitest run`
Expected: exit 0, 전부 PASS.

- [ ] **Step 4: 수동 검증** — 개발/서버에서: 분기를 만들어 분기에서 대화 → 홈 이어하기에 그 분기 미리보기가 뜨고, 클릭 시 그 분기(`/conversations/{분기id}`, 루트 아님)로 진입. 분기 없는 대화는 기존과 동일.

- [ ] **Step 5: 커밋**

```bash
git add app/api/conversations/recent/route.ts "app/(main)/page.tsx"
git commit -m "feat: 홈 이어하기 — 분기 최신 기준 + 분기 직접 진입(/api/conversations/recent)"
```

---

## 배포 (사용자 확인 하에)

```bash
cd apps/web && git push origin HEAD:main
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (홈 이어하기 분기 최신)" && git push origin master
# 서버: git pull origin master && git submodule update --remote apps/web && docker compose up --build -d
```

## Self-Review 메모

- 스펙 커버리지: 스레드 최신 선택(T1), 라우트+isSelected 미리보기+홈 연결(T2) — 전부 매핑.
- 타입 일관성: `ThreadNode`/`pickLatestNodeIdsPerThread`/응답 shape(RecentConv 호환) 일관.
- chatlist·공용 API 불변(회귀 위험 낮음).
