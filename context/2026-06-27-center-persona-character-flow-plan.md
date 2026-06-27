# 센터 페르소나·캐릭터 등록 흐름 통일 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 9개 센터의 채팅 시작/페르소나 흐름을 공용 모듈로 통일하고(기본값·후보·캐릭터 등록), 페르소나 카드 placeholder 치환 방향을 사용자가 고를 수 있게 한다.

**Architecture:** 중복된 센터별 `startChat`/`handlePersonaSelect`를 `lib/centerChat.ts`(순수 후보 빌더 + 대화 생성 래퍼)와 공용 `ChatModeModal`로 추출한다. 페르소나 치환 토글은 `Conversation.personaFlipPlaceholders`에 저장하고 `systemPrompt` 빌더가 매 턴 반영한다.

**Tech Stack:** Next.js 14 App Router, React, Prisma(PostgreSQL, `db push`), Vitest, TypeScript.

설계 문서: `context/2026-06-27-center-persona-character-flow-design.md`

## Global Constraints

- 시스템 프롬프트 조립 순서 불변(CLAUDE.md). 본 작업은 페르소나 블록의 **치환 방식**과 대화 생성 **기본값**만 바꾼다.
- 페르소나 치환 토글 기본값 = **ON(flip)** — 기존 대화방 동작 보존.
- 통일 기본값 = `statsEnabled:true`, `statsConfig:[{name:'호감도',value:50,min:0,max:100}]`, `suggestRepliesEnabled:true`.
- 단위 테스트는 순수 함수만(Vitest). UI/라우트는 `npx tsc --noEmit` 타입체크 + 수동 검증.
- 작업 디렉터리: `apps/web`. 명령은 모두 `apps/web`에서 실행.
- 커밋은 `apps/web`(서브모듈) 단위. 배포(서브모듈 push + 부모 포인터)는 전체 완료 후 사용자 확인 하에.

---

### Task 1: systemPrompt 페르소나 치환 토글 (순수 로직 + 테스트)

**Files:**
- Modify: `lib/systemPrompt.ts` (BuildSystemPromptParams ~15-33, 페르소나 블록 ~144-150; MultiStoryPromptParams ~184-202, 페르소나 블록 ~246-252)
- Test: `lib/systemPrompt.test.ts`

**Interfaces:**
- Produces: `buildStorySystemPrompt`/`buildMultiStorySystemPrompt`가 `flipPersonaPlaceholders?: boolean`(기본 true)를 받아 페르소나 블록 치환 방향을 결정.

- [ ] **Step 1: 실패 테스트 추가** — `lib/systemPrompt.test.ts` 끝에 추가

```ts
describe('페르소나 치환 토글(flipPersonaPlaceholders)', () => {
  const persona = { name: '민수', additionalInfo: '{{char}}는 {{user}}를 짝사랑하는 소꿉친구다' }
  const character = { name: '지영', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' } as any

  it('flip ON(기본): {{char}}→페르소나, {{user}}→AI캐릭터', () => {
    const out = buildStorySystemPrompt({ character, personaCharacter: persona })
    expect(out).toContain('민수는 지영을 짝사랑하는 소꿉친구다')
  })

  it('flip OFF: {{char}}→AI캐릭터, {{user}}→페르소나', () => {
    const out = buildStorySystemPrompt({ character, personaCharacter: persona, flipPersonaPlaceholders: false })
    expect(out).toContain('지영은 민수를 짝사랑하는 소꿉친구다')
  })
})
```

- [ ] **Step 2: 실패 확인**

Run: `npx vitest run lib/systemPrompt.test.ts`
Expected: FAIL — 두 번째 테스트가 "지영은 민수를…"을 못 찾음(현재는 항상 flip).

- [ ] **Step 3: 구현** — `BuildSystemPromptParams`에 `flipPersonaPlaceholders?: boolean` 추가, 함수 시그니처 구조분해에 `flipPersonaPlaceholders = true` 추가. 단일 페르소나 블록 교체:

```ts
  if (personaCharacter) {
    const tagLine = personaCharacter.tags?.length ? `\n태그: ${personaCharacter.tags.join(', ')}` : ''
    const personaInfo = personaCharacter.additionalInfo
      ? (flipPersonaPlaceholders
          ? replacePlaceholders(personaCharacter.additionalInfo, character.name, personaCharacter.name)
          : replacePlaceholders(personaCharacter.additionalInfo, personaCharacter.name, character.name))
      : ''
    parts.push(`[유저 역할]\n이름: ${personaCharacter.name}${tagLine}${personaInfo ? `\n${personaInfo}` : ''}`)
  }
```

`MultiStoryPromptParams`에도 `flipPersonaPlaceholders?: boolean` 추가, 구조분해 `flipPersonaPlaceholders = true` 추가. 멀티 페르소나 블록 교체:

```ts
  if (personaCharacter) {
    const tagLine = personaCharacter.tags?.length ? `\n태그: ${personaCharacter.tags.join(', ')}` : ''
    const charNamesArr = characters.map(c => c.name)
    const personaInfo = personaCharacter.additionalInfo
      ? (flipPersonaPlaceholders
          ? replacePlaceholders(personaCharacter.additionalInfo, charNamesArr.join(', '), personaCharacter.name)
          : replacePlaceholders(personaCharacter.additionalInfo, personaCharacter.name, charNamesArr))
      : ''
    parts.push(`[${personaName} 설정]${tagLine}${personaInfo ? `\n${personaInfo}` : ''}`)
  }
```

- [ ] **Step 4: 통과 확인**

Run: `npx vitest run lib/systemPrompt.test.ts`
Expected: PASS (전체).

- [ ] **Step 5: 커밋**

```bash
git add lib/systemPrompt.ts lib/systemPrompt.test.ts
git commit -m "feat: 페르소나 placeholder 치환 토글(flipPersonaPlaceholders) systemPrompt 지원"
```

---

### Task 2: 스키마 필드 + 대화 생성 시 토글 수용

**Files:**
- Modify: `prisma/schema.prisma` (model Conversation, `personaAutoMode` 근처)
- Modify: `app/api/conversations/route.ts` (POST 본문 파싱·create data)

**Interfaces:**
- Produces: `Conversation.personaFlipPlaceholders: boolean`(기본 true). `POST /api/conversations`가 body의 `personaFlipPlaceholders`를 수용.

- [ ] **Step 1: 스키마 필드 추가** — `model Conversation`의 `personaAutoMode Boolean @default(false)` 아래에:

```prisma
  personaFlipPlaceholders Boolean              @default(true)
```

- [ ] **Step 2: db push 반영**

Run: `npx prisma db push && npx prisma generate`
Expected: "Your database is now in sync with your Prisma schema."

- [ ] **Step 3: POST 라우트 수용** — `app/api/conversations/route.ts` POST에서 conversation `create` data에 추가(다른 불리언 옵션과 같은 자리):

```ts
      personaFlipPlaceholders: typeof body.personaFlipPlaceholders === 'boolean' ? body.personaFlipPlaceholders : true,
```

- [ ] **Step 4: 타입체크**

Run: `npx tsc --noEmit`
Expected: exit 0.

- [ ] **Step 5: 커밋**

```bash
git add prisma/schema.prisma app/api/conversations/route.ts
git commit -m "feat: Conversation.personaFlipPlaceholders 필드 + 대화 생성 수용"
```

---

### Task 3: 프롬프트 빌드 호출부에 토글 연결

**Files:**
- Modify: `lib/chatPipeline.ts` (`buildModeSystemPrompt` ~63-82)
- Modify: `app/api/conversations/[id]/chat/route.ts` (~220-227)
- Modify: `app/api/conversations/[id]/regenerate/route.ts` (~103-110)
- Modify: `app/api/conversations/[id]/continue/route.ts` (~83-90)

**Interfaces:**
- Consumes: Task 1의 `flipPersonaPlaceholders` 빌더 인자.
- Produces: `buildModeSystemPrompt`가 `flipPersonaPlaceholders?: boolean`를 받아 빌더로 전달.

- [ ] **Step 1: buildModeSystemPrompt 인자 추가** — 구조분해 파라미터에 `flipPersonaPlaceholders` 추가, 타입에 `flipPersonaPlaceholders?: boolean` 추가, 두 return 모두에 전달:

```ts
  if (mode === 'multiStory') return buildMultiStorySystemPrompt({ ...base, characters, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders })
  return buildStorySystemPrompt({ ...base, character, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders })
```

- [ ] **Step 2: 3개 라우트에서 전달** — chat·regenerate·continue 각 `buildModeSystemPrompt({...})` 호출에서 `allowPersonaDialogue: conv.personaAutoMode ?? false,` 바로 아래 추가:

```ts
    flipPersonaPlaceholders: conv.personaFlipPlaceholders ?? true,
```

- [ ] **Step 3: 타입체크 + 기존 테스트**

Run: `npx tsc --noEmit && npx vitest run lib/systemPrompt.test.ts`
Expected: exit 0, 테스트 PASS.

- [ ] **Step 4: 커밋**

```bash
git add lib/chatPipeline.ts "app/api/conversations/[id]/chat/route.ts" "app/api/conversations/[id]/regenerate/route.ts" "app/api/conversations/[id]/continue/route.ts"
git commit -m "feat: 프롬프트 빌드에 personaFlipPlaceholders 연결(chat/regenerate/continue)"
```

---

### Task 4: standalone 카드 조회 API

**Files:**
- Modify: `app/api/characters/route.ts` (GET, whereClause 구성부 ~26-29)

**Interfaces:**
- Produces: `GET /api/characters?unassigned=true` → `creatorId=userId AND collectionId=null`인 내 카드만(기존 목록 select 형태) 반환.

- [ ] **Step 1: 쿼리 분기 추가** — `const collectionId = searchParams.get('collectionId')` 아래, `finalWhere` 구성 전에:

```ts
  const unassigned = searchParams.get('unassigned') === 'true'
  const finalWhere = unassigned
    ? { creatorId: userId, collectionId: null }
    : collectionId
      ? { ...whereClause, collectionId }
      : whereClause
```

(기존 `const finalWhere = collectionId ? ... : whereClause` 한 줄을 위 블록으로 대체.)

- [ ] **Step 2: 타입체크**

Run: `npx tsc --noEmit`
Expected: exit 0.

- [ ] **Step 3: 수동 검증** — 개발 서버에서 로그인 후 `/api/characters?unassigned=true` 호출 시 `collectionId`가 없는(복제/직접생성) 카드만 오는지 확인.

- [ ] **Step 4: 커밋**

```bash
git add app/api/characters/route.ts
git commit -m "feat: GET /api/characters?unassigned=true (standalone 카드 조회)"
```

---

### Task 5: 페르소나 후보 빌더 (순수 로직 + 테스트)

**Files:**
- Create: `lib/centerChat.ts`
- Test: `lib/centerChat.test.ts`

**Interfaces:**
- Produces:
  - 타입 `PersonaCandidate = { id: string; name: string; gender: string; avatarUrl: string | null }`
  - `buildPersonaCandidates(args: { collectionChars: PersonaCandidate[]; standaloneCards: PersonaCandidate[]; aiCharIds: string[] }): PersonaCandidate[]` — 멀티 동료 + standalone 병합, `aiCharIds` 제외, id 기준 중복 제거(앞선 항목 우선).

- [ ] **Step 1: 실패 테스트** — `lib/centerChat.test.ts`

```ts
import { describe, it, expect } from 'vitest'
import { buildPersonaCandidates } from './centerChat'

const c = (id: string, name = id) => ({ id, name, gender: '', avatarUrl: null })

describe('buildPersonaCandidates', () => {
  it('AI 캐릭터는 후보에서 제외한다', () => {
    const out = buildPersonaCandidates({ collectionChars: [c('a'), c('b')], standaloneCards: [], aiCharIds: ['a'] })
    expect(out.map(x => x.id)).toEqual(['b'])
  })

  it('멀티 동료 + standalone를 합치고 id 중복은 제거한다', () => {
    const out = buildPersonaCandidates({ collectionChars: [c('b')], standaloneCards: [c('b'), c('d')], aiCharIds: ['a'] })
    expect(out.map(x => x.id)).toEqual(['b', 'd'])
  })
})
```

- [ ] **Step 2: 실패 확인**

Run: `npx vitest run lib/centerChat.test.ts`
Expected: FAIL — 모듈/함수 없음.

- [ ] **Step 3: 구현** — `lib/centerChat.ts`

```ts
export type PersonaCandidate = { id: string; name: string; gender: string; avatarUrl: string | null }

export function buildPersonaCandidates(args: {
  collectionChars: PersonaCandidate[]
  standaloneCards: PersonaCandidate[]
  aiCharIds: string[]
}): PersonaCandidate[] {
  const exclude = new Set(args.aiCharIds)
  const seen = new Set<string>()
  const out: PersonaCandidate[] = []
  for (const cand of [...args.collectionChars, ...args.standaloneCards]) {
    if (exclude.has(cand.id) || seen.has(cand.id)) continue
    seen.add(cand.id)
    out.push(cand)
  }
  return out
}
```

- [ ] **Step 4: 통과 확인**

Run: `npx vitest run lib/centerChat.test.ts`
Expected: PASS.

- [ ] **Step 5: 커밋**

```bash
git add lib/centerChat.ts lib/centerChat.test.ts
git commit -m "feat: buildPersonaCandidates (멀티 동료 + standalone 병합)"
```

---

### Task 6: 대화 생성 래퍼 createCenterChat

**Files:**
- Modify: `lib/centerChat.ts`

**Interfaces:**
- Consumes: `api`(`@/lib/api`), Task 2의 POST 필드.
- Produces:
  - `NewPersonaData = { name: string; gender: string; additionalInfo: string }` (재노출 또는 `@/components/ui/WhifPersonaModal`에서 import)
  - `createCenterChat(args: { collectionId: string; title: string; aiCharIds: string[]; personaCharId: string | null; newPersona?: NewPersonaData; flipPlaceholders: boolean; opening?: string; extras?: Record<string, unknown> }): Promise<{ id: string }>`
    — newPersona면 먼저 `POST /api/characters`(collectionId 포함)로 생성. 그 후 `POST /api/conversations`에 통일 기본값 + 페르소나 + flip + extras 병합. 반환은 생성된 conversation.

- [ ] **Step 1: import는 파일 최상단에 추가** — `lib/centerChat.ts` 맨 위에:

```ts
import { api } from '@/lib/api'
import type { NewPersonaData } from '@/components/ui/WhifPersonaModal'
export type { NewPersonaData }
```

그 다음, 파일 하단(`buildPersonaCandidates` 아래)에 나머지를 추가:

```ts
const DEFAULT_CHAT_OPTIONS = {
  statsEnabled: true,
  statsConfig: [{ name: '호감도', value: 50, min: 0, max: 100 }],
  suggestRepliesEnabled: true,
}

export async function createCenterChat(args: {
  collectionId: string
  title: string
  aiCharIds: string[]
  personaCharId: string | null
  newPersona?: NewPersonaData
  flipPlaceholders: boolean
  opening?: string
  extras?: Record<string, unknown>
}): Promise<{ id: string }> {
  let personaId = args.personaCharId
  if (!personaId && args.newPersona) {
    const p = await api.post('/api/characters', {
      name: args.newPersona.name,
      gender: args.newPersona.gender,
      additionalInfo: args.newPersona.additionalInfo,
      collectionId: args.collectionId,
    })
    personaId = p.id
  }
  return api.post('/api/conversations', {
    title: args.title,
    characterIds: args.aiCharIds,
    mode: args.aiCharIds.length > 1 ? 'multiStory' : 'story',
    personaCharacterId: personaId,
    personaFlipPlaceholders: args.flipPlaceholders,
    ...DEFAULT_CHAT_OPTIONS,
    ...(args.opening?.trim() ? { openingMessage: args.opening } : {}),
    ...(args.extras ?? {}),
  })
}
```

- [ ] **Step 2: 타입체크**

Run: `npx tsc --noEmit`
Expected: exit 0. (WhifPersonaModal이 `NewPersonaData`를 export 중인지 확인 — 이미 export됨.)

- [ ] **Step 3: 커밋**

```bash
git add lib/centerChat.ts
git commit -m "feat: createCenterChat (통일 기본값 + 페르소나 + flip 대화 생성)"
```

---

### Task 7: WhifPersonaModal 치환 토글 추가

**Files:**
- Modify: `components/ui/WhifPersonaModal.tsx`

**Interfaces:**
- Consumes: 기존 `Props`.
- Produces: `onSelect` 시그니처 확장 → `onSelect(personaCharId: string | null, newPersona: NewPersonaData | undefined, flipPlaceholders: boolean)`. 새 prop `defaultFlip?: boolean`(기본 true). 모달 하단에 "이 캐릭터의 설정을 페르소나 기준으로 치환" 토글 표시.

- [ ] **Step 1: 상태·prop 추가** — `Props`에 `defaultFlip?: boolean` 추가, `onSelect` 타입을 위 시그니처로 변경. 컴포넌트 내부 상태 추가:

```ts
  const [flip, setFlip] = useState(defaultFlip ?? true)
```

- [ ] **Step 2: 토글 UI 추가** — 시작 버튼(`handleStart`) 위, 폼 영역 하단에:

```tsx
        <label style={{ display: 'flex', alignItems: 'center', gap: 8, fontSize: 12, color: 'var(--w-ink-soft)', cursor: 'pointer', marginTop: 10 }}>
          <input type="checkbox" checked={flip} onChange={e => setFlip(e.target.checked)} />
          페르소나 카드의 설정을 페르소나 기준으로 치환 ({'{{char}}'}→페르소나, {'{{user}}'}→캐릭터)
        </label>
```

- [ ] **Step 3: onSelect 호출에 flip 전달** — `handleStart` 내부 두 `onSelect(...)` 호출 수정:

```ts
  const handleStart = () => {
    if (tab === 'existing' && selectedId) {
      onSelect(selectedId, undefined, flip)
    } else {
      const additionalInfo = [
        settings.trim() && `성격/설정: ${settings.trim()}`,
        relationships.length > 0 && `관계: ${relationships.join(', ')}`,
      ].filter(Boolean).join('\n')
      onSelect(null, { name: name.trim() || '유저', gender, additionalInfo }, flip)
    }
  }
```

- [ ] **Step 4: 타입체크** — 이 시점에서 9개 센터의 `onSelect` 콜백이 새 3번째 인자로 인해 타입 에러가 날 수 있음. 다음 태스크들에서 콜백을 교체하므로, 본 태스크는 모달 파일만 검증:

Run: `npx tsc --noEmit 2>&1 | grep WhifPersonaModal.tsx || echo "modal OK"`
Expected: "modal OK" (모달 자체 오류 없음). 호출부 오류는 Task 8/9에서 해소.

- [ ] **Step 5: 커밋**

```bash
git add components/ui/WhifPersonaModal.tsx
git commit -m "feat: WhifPersonaModal 페르소나 치환 토글 + onSelect flip 인자"
```

---

### Task 8: ChatModeModal 컴포넌트 추출

**Files:**
- Create: `components/ui/ChatModeModal.tsx`
- Modify: `app/(zeta)/zeta/plots/[id]/page.tsx` (인라인 chatMode 모달 ~178-202를 컴포넌트로 교체)

**Interfaces:**
- Produces: `ChatModeModal({ characters, onPick, onClose })` — `characters: {id:string;name:string;avatarUrl:string|null}[]`. "전체와 대화" → `onPick(characters.map(c=>c.id))`, 각 캐릭터 "1:1" → `onPick([id])`.

- [ ] **Step 1: 컴포넌트 작성** — zeta 인라인 모달의 마크업을 일반 테마 변수로 옮겨 작성:

```tsx
'use client'
interface Props {
  characters: { id: string; name: string; avatarUrl: string | null }[]
  onPick: (aiCharIds: string[]) => void
  onClose: () => void
}
export default function ChatModeModal({ characters, onPick, onClose }: Props) {
  return (
    <div onClick={e => { if (e.target === e.currentTarget) onClose() }}
      style={{ position: 'fixed', inset: 0, background: 'rgba(0,0,0,0.5)', display: 'grid', placeItems: 'center', zIndex: 100 }}>
      <div style={{ background: 'var(--pane)', borderRadius: 'var(--radius-lg)', padding: 16, width: 'min(360px, 90vw)', display: 'flex', flexDirection: 'column', gap: 8 }}>
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
          <div style={{ fontWeight: 800 }}>누구와 대화할까요?</div>
          <button onClick={onClose} style={{ background: 'none', border: 'none', fontSize: 18, cursor: 'pointer', color: 'var(--ink-soft)' }}>×</button>
        </div>
        <button onClick={() => onPick(characters.map(c => c.id))}
          style={{ appearance: 'none', border: '1px solid var(--hairline)', background: 'var(--accent)', color: '#fff', borderRadius: 'var(--radius)', padding: '10px', cursor: 'pointer', fontWeight: 700 }}>
          전체와 대화 (멀티)
        </button>
        {characters.map(c => (
          <button key={c.id} onClick={() => onPick([c.id])}
            style={{ appearance: 'none', border: '1px solid var(--hairline)', background: 'var(--bg-2)', color: 'var(--ink)', borderRadius: 'var(--radius)', padding: '10px', cursor: 'pointer', display: 'flex', alignItems: 'center', gap: 8 }}>
            {c.avatarUrl ? <img src={c.avatarUrl} loading="lazy" decoding="async" alt="" style={{ width: 28, height: 28, borderRadius: 6, objectFit: 'cover' }} /> : <div style={{ width: 28, height: 28, borderRadius: 6, background: 'var(--hairline)' }} />}
            {c.name}와 1:1
          </button>
        ))}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: zeta에서 사용** — zeta 페이지의 인라인 chatMode 모달(`{chatModeOpen && (...)}`)을 다음으로 교체:

```tsx
      {chatModeOpen && (
        <ChatModeModal
          characters={col.characters.map(c => ({ id: c.id, name: c.name, avatarUrl: c.avatarUrl }))}
          onClose={() => setChatModeOpen(false)}
          onPick={(ids) => { setPendingAiCharIds(ids); setChatModeOpen(false); setPersonaOpen(true) }}
        />
      )}
```

상단에 `import ChatModeModal from '@/components/ui/ChatModeModal'` 추가.

- [ ] **Step 3: 타입체크**

Run: `npx tsc --noEmit 2>&1 | grep -E "ChatModeModal|zeta/plots" || echo "OK"`
Expected: "OK"(zeta는 아직 onSelect flip 인자 미수정이라 Task 9에서 함께 해소되지만, ChatModeModal 자체 오류는 없어야 함).

- [ ] **Step 4: 커밋**

```bash
git add components/ui/ChatModeModal.tsx "app/(zeta)/zeta/plots/[id]/page.tsx"
git commit -m "feat: ChatModeModal 추출 + zeta 적용"
```

---

### Task 9: 센터 상세 페이지 9개 통일 연결

각 센터 카드 상세 페이지를 공용 모듈로 연결한다. **공통 변경**(모든 센터 동일):

1. import 추가: `import { createCenterChat, buildPersonaCandidates, type PersonaCandidate } from '@/lib/centerChat'`, `import ChatModeModal from '@/components/ui/ChatModeModal'`.
2. standalone 후보 상태/조회 추가:

```tsx
  const [standalone, setStandalone] = useState<PersonaCandidate[]>([])
  useEffect(() => { api.get('/api/characters?unassigned=true').then(setStandalone).catch(() => {}) }, [])
```

3. `startChat`: `col.characters.length > 1`이면 `setChatModeOpen(true)`, 아니면 단일 캐릭터 id로 `setPendingAiCharIds([...]); setPersonaOpen(true)` (zeta 패턴). chatMode/pendingAiCharIds 상태가 없으면 추가.
4. `personaCandidates` 계산:

```tsx
  const aiCharIds = pendingAiCharIds ?? (mainChar ? [mainChar.id] : [])
  const personaCandidates = buildPersonaCandidates({
    collectionChars: col.characters.map(c => ({ id: c.id, name: c.name, gender: c.gender || '', avatarUrl: c.avatarUrl })),
    standaloneCards: standalone,
    aiCharIds,
  })
```

5. `handlePersonaSelect`를 공용 래퍼로 교체:

```tsx
  const handlePersonaSelect = async (personaCharId: string | null, newPersona?: NewPersonaData, flip = true) => {
    if (!mainChar) return
    setCreating(true); setError('')
    try {
      const resp = await createCenterChat({
        collectionId: col.id,
        title: col.title,
        aiCharIds,
        personaCharId,
        newPersona,
        flipPlaceholders: flip,
        opening: /* 센터별 도입부 변수, 없으면 생략 */ undefined,
        extras: /* 아래 표의 센터별 extras */ {},
      })
      router.push(`/conversations/${resp.id}`)
    } catch (e: any) { setError('채팅방 생성 실패: ' + e.message); setCreating(false) }
  }
```

6. `<WhifPersonaModal candidates={personaCandidates} ... />` — `candidates={[]}`였던 센터는 `personaCandidates`로 교체. `onSelect={handlePersonaSelect}`는 이미 3-인자 시그니처와 호환(flip 기본값 처리).
7. `ChatModeModal` 렌더 추가(Task 8 zeta와 동일 패턴).
8. "캐릭터" 섹션 헤더에 등록 버튼 추가:

```tsx
  <button onClick={() => router.push(`/characters/new?is<Center>=true&collectionId=${col.id}`)}>+ 캐릭터 등록</button>
```

**센터별 차이 표** (위 `<Center>` 파라미터 / extras / opening / 테마 칩):

| 센터 | is파라미터 | extras | opening 변수 |
|---|---|---|---|
| zeta | isZeta | `{ ...(col.description ? { scenarioDescription: col.description } : {}) }` | 선택한 `openings[openingIdx]?.content` |
| whif | isWhif | `{}` | 없음 |
| melting | isMelting | `{}` | `opening` |
| tikita | isTikita | `{ ...(introSections['배경'] ? { scenarioDescription: introSections['배경'] } : {}) }` | 도입부 변수 |
| chub | isChub | `{}` | `opening` |
| rofan | isRofan | `{}` | `opening` |
| loveydovey | isLoveydovey | `{}` | `opening` |
| babechat | isBabechat | `{}` | `opening` |
| tingle | isTingle | `{}` | `opening` |

> melting/rofan/babechat/loveydovey/chub은 기존에 `statsEnabled`·`suggestRepliesEnabled`를 직접 넘기던 코드를 **삭제**(이제 `createCenterChat` 기본값이 담당). tikita의 `suggestRepliesEnabled: true`, zeta의 multi/story·scenario 처리도 `createCenterChat`/extras로 이전.

**Files (수정):**
- `app/(zeta)/zeta/plots/[id]/page.tsx`
- `app/(whif)/whif/characters/[id]/page.tsx`
- `app/(melting)/melting/characters/[id]/page.tsx`
- `app/(tikita)/tikita/story/[id]/page.tsx`
- `app/(chub)/chub/characters/[id]/page.tsx`
- `app/(rofan)/rofan/characters/[id]/page.tsx`
- `app/(loveydovey)/loveydovey/characters/[id]/page.tsx`
- `app/(babechat)/babechat/characters/[id]/page.tsx`
- `app/(tingle)/tingle/characters/[id]/page.tsx`

- [ ] **Step 1: zeta 마무리** — Task 8에서 ChatModeModal 적용됨. 위 공통 변경 4·5·6(standalone 후보, createCenterChat, flip 인자)을 zeta에 적용.

- [ ] **Step 2: 나머지 8개 센터에 동일 패턴 적용** — 위 공통 변경 + 표의 센터별 값으로 각 페이지 수정. 기존 직접 `statsEnabled`/`suggestRepliesEnabled` 라인 제거.

- [ ] **Step 3: 타입체크 (전체)**

Run: `npx tsc --noEmit`
Expected: exit 0 (모든 onSelect/모달 호출 정합).

- [ ] **Step 4: 수동 검증(센터 1곳)** — 개발 서버에서 한 센터 카드 → "+ 캐릭터 등록" → 등록 → 채팅 시작 → ChatModeModal → 페르소나 모달에 standalone/동료 후보 노출 + 치환 토글 표시 → 대화 생성 후 호감도 스탯·추천답변 켜져 있는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add "app/(zeta)/zeta/plots/[id]/page.tsx" "app/(whif)/whif/characters/[id]/page.tsx" "app/(melting)/melting/characters/[id]/page.tsx" "app/(tikita)/tikita/story/[id]/page.tsx" "app/(chub)/chub/characters/[id]/page.tsx" "app/(rofan)/rofan/characters/[id]/page.tsx" "app/(loveydovey)/loveydovey/characters/[id]/page.tsx" "app/(babechat)/babechat/characters/[id]/page.tsx" "app/(tingle)/tingle/characters/[id]/page.tsx"
git commit -m "feat: 9개 센터 채팅 시작/페르소나/캐릭터 등록 흐름 통일"
```

---

### Task 10: 기능 가이드 동기화 + 최종 검증

**Files:**
- Modify: `app/(main)/guide/page.tsx` (FEATURE_SECTIONS)

- [ ] **Step 1: 가이드 항목 추가** — 적절한 섹션(예: 캐릭터/페르소나 관련)에 항목 추가:

```ts
      { emoji: '🎭', label: '캐릭터 등록·페르소나 선택', desc: '모든 센터 카드에서 + 캐릭터 등록으로 캐릭터를 추가해 멀티로 만들 수 있고, 채팅 시작 시 기존 캐릭터(동료·내 카드)를 페르소나로 고르거나 새로 입력할 수 있습니다. 기존 캐릭터를 페르소나로 쓸 때 설정의 {{char}}/{{user}} 치환 방향을 토글로 선택합니다.' },
```

- [ ] **Step 2: 전체 검증**

Run: `npx tsc --noEmit && npx vitest run`
Expected: exit 0, 전체 테스트 PASS.

- [ ] **Step 3: 커밋**

```bash
git add "app/(main)/guide/page.tsx"
git commit -m "docs: 캐릭터 등록·페르소나 선택 기능 가이드 동기화"
```

---

## 배포 (전체 완료 후, 사용자 확인 하에)

```bash
# 1. 서브모듈 push
cd apps/web && git push origin HEAD:main
# 2. 부모 포인터 갱신
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (센터 페르소나·캐릭터 등록 흐름 통일)" && git push origin master
```

## Self-Review 메모

- 스펙 커버리지: 기본값 통일(Task 6·9), 페르소나 모달 통일(Task 5·7·9), 캐릭터 등록(Task 9), 치환 토글(Task 1·2·3·7·9), unassigned API(Task 4), guide(Task 10) — 전부 매핑됨.
- ⚠️ 구현 시 주의: Task 9의 센터별 페이지는 현재 코드 구조(상태 변수명·도입부 변수명)가 조금씩 다르므로, 각 파일에서 `handlePersonaSelect`·`startChat`·모달 렌더의 **실제 현재 코드**를 먼저 확인하고 위 패턴으로 치환할 것. tikita의 `introSections` 키('배경' 등)는 실제 파싱 키를 확인해 extras에 매핑.
