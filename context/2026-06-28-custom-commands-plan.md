# 사용자 정의 AI 커맨드 + 커맨드 응답 마크다운 렌더링 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 사용자가 채팅 설정창에서 `!에타` 같은 AI 프롬프트 매크로 커맨드를 만들어 재사용하고, 커맨드 응답(커스텀·빌트인)을 마크다운으로 렌더하며, 빌트인 상태 커맨드는 값이 없으면 AI가 마크다운 상태창을 생성하도록 한다.

**Architecture:** 순수 커맨드 로직(`lib/commands.ts`)을 분리해 TDD하고, 채팅 라우트의 기존 "커맨드 엔진"을 확장해 (a) 빌트인 결정적 출력, (b) 빌트인 빈값→AI 폴백, (c) 커스텀→AI 생성을 분기한다. AI 경로는 기존 생성 파이프라인(`systemPrompt` 문자열에 이번 턴 한정 지시 블록을 덧붙임)을 그대로 재사용한다. 메시지에 `commandName` 플래그를 달아 렌더러가 `react-markdown`으로 분기한다.

**Tech Stack:** Next.js 14 (App Router), Prisma/PostgreSQL, React, vitest(node, `lib/**/*.test.ts`), 신규 의존성 `react-markdown`·`remark-gfm`.

## Global Constraints

- 테스트: `npm test`(=`vitest run`), env `node`, include `lib/**/*.test.ts`. 순수 로직만 단위 테스트, React/route/Prisma는 `npm run build`(타입체크)+수동.
- 시스템 프롬프트 고정 순서(globalRules→…→closingRules) 유지. 커맨드 지시는 **이번 생성 한정으로 systemPrompt 문자열 끝에 덧붙임**(DB 영속 X, 순서 자체는 불변).
- 커맨드 범위: **계정별(글로벌)**. `UserCommand @@unique([userId, name])`.
- `name`은 `!` 제외. 빌트인 예약어와 중복 금지. 빈 instruction 금지.
- 마크다운 렌더는 `react-markdown`의 안전 모드(원시 HTML 미렌더). 커스텀+빌트인 커맨드 응답 모두 적용.
- 작업 디렉터리 `/home/server/StoryFit/apps/web`. 별도 feature 브랜치 권장. 사용자 체감 기능이므로 `app/(main)/guide/page.tsx` 동기화(Task 8).
- 경로 alias `@` = apps/web 루트.

---

## File Structure

- 신규: `lib/commands.ts`(+`lib/commands.test.ts`) — 순수 파싱/분류/지시합성.
- 신규: `app/api/commands/route.ts`, `app/api/commands/[id]/route.ts` — CRUD.
- 신규: `components/ui/MarkdownText.tsx` — react-markdown 래퍼.
- 신규: `components/ui/CommandGuide.tsx` — 지시문 작성 가이드(? 도움말) 콘텐츠.
- 수정: `prisma/schema.prisma` — `UserCommand` 모델 + `Message.commandName` + `User.userCommands` 관계.
- 수정: `app/api/conversations/[id]/chat/route.ts` — 커맨드 엔진 확장(분기 + 지시 주입 + commandName 태깅).
- 수정: `app/(main)/conversations/[id]/_components/MessageList.tsx` — commandName이면 MarkdownText 렌더.
- 수정: `app/(main)/conversations/[id]/_components/SidePanel.tsx` — "내 커맨드" CRUD + ? 가이드.
- 수정: `app/(main)/conversations/[id]/page.tsx` + `_lib/chatShared.ts` — 자동완성에 유저 커맨드 병합.
- 수정: `app/(main)/guide/page.tsx` — FEATURE_SECTIONS 항목 추가.

---

## Task 1: 순수 커맨드 로직 (`lib/commands.ts`)

**Files:**
- Create: `lib/commands.ts`
- Test: `lib/commands.test.ts`

**Interfaces:**
- Produces:
  - `interface ParsedCommand { name: string; args: string }`
  - `function parseCommand(text: string): ParsedCommand | null`
  - `function isBuiltinCommand(name: string): boolean`
  - `function builtinFallbackKey(name: string): 'status' | 'stats' | 'inventory' | 'scene' | null`
  - `const BUILTIN_FALLBACK: Record<'status'|'stats'|'inventory'|'scene', string>`
  - `function composeCommandDirective(name: string, instruction: string, args: string): string`
  - `function validateCommandName(name: string): string | null` (null=유효, 아니면 에러 메시지)

- [ ] **Step 1: 실패 테스트 작성**

`lib/commands.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { parseCommand, isBuiltinCommand, builtinFallbackKey, composeCommandDirective, validateCommandName } from './commands'

describe('parseCommand', () => {
  it('이름과 인자를 분리한다', () => {
    expect(parseCommand('!에타 오늘 사건 위주로')).toEqual({ name: '에타', args: '오늘 사건 위주로' })
  })
  it('인자 없으면 args는 빈 문자열', () => {
    expect(parseCommand('  !상태창  ')).toEqual({ name: '상태창', args: '' })
  })
  it('! 로 시작하지 않으면 null', () => {
    expect(parseCommand('안녕')).toBeNull()
  })
})

describe('isBuiltinCommand', () => {
  it('예약어는 true(대소문자 무관)', () => {
    expect(isBuiltinCommand('상태창')).toBe(true)
    expect(isBuiltinCommand('STATUS')).toBe(true)
    expect(isBuiltinCommand('도움말')).toBe(true)
  })
  it('그 외는 false', () => {
    expect(isBuiltinCommand('에타')).toBe(false)
  })
})

describe('builtinFallbackKey', () => {
  it('상태계 별칭을 키로 매핑, 도움말은 null', () => {
    expect(builtinFallbackKey('정보')).toBe('status')
    expect(builtinFallbackKey('호감도')).toBe('stats')
    expect(builtinFallbackKey('소지품')).toBe('inventory')
    expect(builtinFallbackKey('타임라인')).toBe('scene')
    expect(builtinFallbackKey('도움말')).toBeNull()
    expect(builtinFallbackKey('에타')).toBeNull()
  })
})

describe('composeCommandDirective', () => {
  it('지시문+인자+마크다운 지시를 합성', () => {
    const d = composeCommandDirective('에타', '게시글 작성', '오늘 위주')
    expect(d).toContain('[사용자 커맨드: 에타]')
    expect(d).toContain('게시글 작성')
    expect(d).toContain('추가 지시: 오늘 위주')
    expect(d).toContain('마크다운')
  })
  it('인자 없으면 추가 지시 줄 없음', () => {
    expect(composeCommandDirective('에타', '게시글 작성', '')).not.toContain('추가 지시:')
  })
})

describe('validateCommandName', () => {
  it('빌트인/중복 예약어는 에러', () => {
    expect(validateCommandName('상태창')).not.toBeNull()
  })
  it('빈 값·공백·! 포함은 에러', () => {
    expect(validateCommandName('')).not.toBeNull()
    expect(validateCommandName('에 타')).not.toBeNull()
    expect(validateCommandName('!에타')).not.toBeNull()
  })
  it('정상 이름은 null', () => {
    expect(validateCommandName('에타')).toBeNull()
  })
})
```

- [ ] **Step 2: 실패 확인**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/commands.test.ts`
Expected: FAIL — `parseCommand` 등 미정의.

- [ ] **Step 3: 구현**

`lib/commands.ts`:

```ts
// 채팅 커맨드(!xxx)의 순수 파싱/분류/지시합성. UI/DB 무관.

export interface ParsedCommand { name: string; args: string }

export function parseCommand(text: string): ParsedCommand | null {
  const t = text.trim()
  if (!t.startsWith('!')) return null
  const rest = t.slice(1)
  const m = rest.match(/^(\S+)\s*([\s\S]*)$/)
  if (!m) return null
  return { name: m[1], args: m[2].trim() }
}

// 빌트인 예약어(소문자). 채팅 라우트의 결정적 커맨드 + 폴백 대상.
const BUILTIN_NAMES = new Set([
  '상태창', '정보', 'status',
  '스탯', '능력치', 'stats', '호감도', '관계',
  '인벤토리', '소지품', '인벤', 'inventory',
  '상황', '씬', 'scene', '타임라인',
  '도움말', '명령어', 'help',
])

export function isBuiltinCommand(name: string): boolean {
  return BUILTIN_NAMES.has(name.toLowerCase())
}

export function builtinFallbackKey(name: string): 'status' | 'stats' | 'inventory' | 'scene' | null {
  const n = name.toLowerCase()
  if (['상태창', '정보', 'status'].includes(n)) return 'status'
  if (['스탯', '능력치', 'stats', '호감도', '관계'].includes(n)) return 'stats'
  if (['인벤토리', '소지품', '인벤', 'inventory'].includes(n)) return 'inventory'
  if (['상황', '씬', 'scene', '타임라인'].includes(n)) return 'scene'
  return null // 도움말 등 폴백 없음
}

export const BUILTIN_FALLBACK: Record<'status' | 'stats' | 'inventory' | 'scene', string> = {
  status: '현재 상황(직전 대화·타임라인·기존 스탯/인벤토리)을 근거로 캐릭터의 스탯·호감도·소지품·현재 상황을 합리적으로 추론해, 마크다운 형식의 상태창으로 작성하라.',
  stats: '현재 상황을 근거로 캐릭터의 능력치와 관계(호감도) 스탯을 합리적으로 추론해, 마크다운 형식으로 작성하라.',
  inventory: '현재 상황을 근거로 보유 중일 법한 소지품(인벤토리)을 합리적으로 추론해, 마크다운 목록으로 작성하라.',
  scene: '현재 상황(시간대·장소·복장·분위기 등)을 마크다운 형식의 상황 요약으로 작성하라.',
}

export function composeCommandDirective(name: string, instruction: string, args: string): string {
  const extra = args ? `\n추가 지시: ${args}` : ''
  return `[사용자 커맨드: ${name}]\n${instruction}${extra}\n\n응답은 마크다운 형식으로 작성하라.`
}

// 커맨드 이름 검증. null=유효, 문자열=에러 사유.
export function validateCommandName(name: string): string | null {
  const n = (name ?? '').trim()
  if (!n) return '이름을 입력하세요.'
  if (/\s/.test(n) || n.includes('!')) return '이름에는 공백이나 ! 를 넣을 수 없습니다.'
  if (n.length > 20) return '이름은 20자 이내여야 합니다.'
  if (!/^[가-힣a-zA-Z0-9_]+$/.test(n)) return '이름은 한글·영문·숫자·_ 만 가능합니다.'
  if (isBuiltinCommand(n)) return '기본 명령어와 같은 이름은 쓸 수 없습니다.'
  return null
}
```

- [ ] **Step 4: 통과 확인**

Run: `cd /home/server/StoryFit/apps/web && npx vitest run lib/commands.test.ts`
Expected: PASS.

- [ ] **Step 5: 전체 스위트**

Run: `cd /home/server/StoryFit/apps/web && npm test`
Expected: 기존 + 신규 전부 PASS.

- [ ] **Step 6: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add lib/commands.ts lib/commands.test.ts && git commit -m "feat: 순수 커맨드 로직(parse/classify/directive)"
```

---

## Task 2: Prisma — UserCommand 모델 + Message.commandName

**Files:**
- Modify: `prisma/schema.prisma`

**Interfaces:**
- Produces: `prisma.userCommand` 모델(`id, userId, name, instruction, description?, createdAt`, unique `[userId,name]`), `Message.commandName String?`.

- [ ] **Step 1: 스키마 수정 — UserCommand 모델 추가**

`prisma/schema.prisma`의 `model AdminLog` 위(또는 파일 끝 근처)에 추가:

```prisma
model UserCommand {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  name        String
  instruction String
  description String   @default("")
  createdAt   DateTime @default(now())

  @@unique([userId, name])
  @@index([userId])
}
```

- [ ] **Step 2: User 관계 추가**

`model User`의 관계 목록(`promptPresets PromptPreset[]` 줄 아래)에 추가:

```prisma
  userCommands  UserCommand[]
```

- [ ] **Step 3: Message에 commandName 추가**

`model Message`의 `bookmarked Boolean @default(false)` 줄 아래에 추가:

```prisma
  commandName    String?
```

- [ ] **Step 4: 마이그레이션 생성·적용 + 클라이언트 재생성**

Run:
```bash
cd /home/server/StoryFit/apps/web && npx prisma migrate dev --name add_user_commands_and_message_commandname && npx prisma generate
```
Expected: 마이그레이션 SQL 생성·적용, Prisma Client 재생성 성공.
(DB 접속 불가 환경이면 `npx prisma migrate dev`가 실패할 수 있음 → 그 경우 DB 접속 가능한 환경에서 실행해야 하며, 최소 `npx prisma generate`로 타입은 갱신. 보고에 상황 명시.)

- [ ] **Step 5: 타입체크**

Run: `cd /home/server/StoryFit/apps/web && npx tsc --noEmit`
Expected: exit 0.

- [ ] **Step 6: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add prisma/schema.prisma prisma/migrations && git commit -m "feat: UserCommand 모델 + Message.commandName 마이그레이션"
```

---

## Task 3: API `/api/commands` CRUD

**Files:**
- Create: `app/api/commands/route.ts`
- Create: `app/api/commands/[id]/route.ts`

**Interfaces:**
- Consumes: `validateCommandName`(Task 1), `prisma.userCommand`(Task 2), `authenticate`.
- Produces: `GET /api/commands`→`UserCommand[]`, `POST`(create), `PATCH /api/commands/[id]`, `DELETE /api/commands/[id]`.

- [ ] **Step 1: 목록·생성 라우트**

`app/api/commands/route.ts`:

```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { authenticate } from '@/lib/apiAuth'
import { validateCommandName } from '@/lib/commands'

export async function GET(req: NextRequest) {
  const userId = await authenticate(req)
  if (!userId) return NextResponse.json({ error: '인증이 필요합니다.' }, { status: 401 })
  const commands = await prisma.userCommand.findMany({
    where: { userId },
    orderBy: { createdAt: 'asc' },
  })
  return NextResponse.json(commands)
}

export async function POST(req: NextRequest) {
  const userId = await authenticate(req)
  if (!userId) return NextResponse.json({ error: '인증이 필요합니다.' }, { status: 401 })
  const { name, instruction, description } = await req.json()
  const nameErr = validateCommandName(name)
  if (nameErr) return NextResponse.json({ error: nameErr }, { status: 400 })
  if (!instruction?.trim()) return NextResponse.json({ error: '지시문을 입력하세요.' }, { status: 400 })
  const dup = await prisma.userCommand.findUnique({ where: { userId_name: { userId, name: name.trim() } } })
  if (dup) return NextResponse.json({ error: '같은 이름의 커맨드가 이미 있습니다.' }, { status: 400 })
  const created = await prisma.userCommand.create({
    data: { userId, name: name.trim(), instruction: instruction.trim(), description: (description ?? '').trim() },
  })
  return NextResponse.json(created, { status: 201 })
}
```

- [ ] **Step 2: 수정·삭제 라우트**

`app/api/commands/[id]/route.ts`:

```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { authenticate } from '@/lib/apiAuth'
import { validateCommandName } from '@/lib/commands'

export async function PATCH(req: NextRequest, { params }: { params: { id: string } }) {
  const userId = await authenticate(req)
  if (!userId) return NextResponse.json({ error: '인증이 필요합니다.' }, { status: 401 })
  const existing = await prisma.userCommand.findUnique({ where: { id: params.id } })
  if (!existing || existing.userId !== userId) return NextResponse.json({ error: '찾을 수 없습니다.' }, { status: 404 })
  const { name, instruction, description } = await req.json()
  const data: { name?: string; instruction?: string; description?: string } = {}
  if (name !== undefined) {
    const nameErr = validateCommandName(name)
    if (nameErr) return NextResponse.json({ error: nameErr }, { status: 400 })
    if (name.trim() !== existing.name) {
      const dup = await prisma.userCommand.findUnique({ where: { userId_name: { userId, name: name.trim() } } })
      if (dup) return NextResponse.json({ error: '같은 이름의 커맨드가 이미 있습니다.' }, { status: 400 })
    }
    data.name = name.trim()
  }
  if (instruction !== undefined) {
    if (!instruction.trim()) return NextResponse.json({ error: '지시문을 입력하세요.' }, { status: 400 })
    data.instruction = instruction.trim()
  }
  if (description !== undefined) data.description = description.trim()
  const updated = await prisma.userCommand.update({ where: { id: params.id }, data })
  return NextResponse.json(updated)
}

export async function DELETE(req: NextRequest, { params }: { params: { id: string } }) {
  const userId = await authenticate(req)
  if (!userId) return NextResponse.json({ error: '인증이 필요합니다.' }, { status: 401 })
  const existing = await prisma.userCommand.findUnique({ where: { id: params.id } })
  if (!existing || existing.userId !== userId) return NextResponse.json({ error: '찾을 수 없습니다.' }, { status: 404 })
  await prisma.userCommand.delete({ where: { id: params.id } })
  return NextResponse.json({ ok: true })
}
```

- [ ] **Step 3: 빌드(타입체크)**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: Compiled successfully (타입 에러 없음). Prisma Client에 `userCommand`가 있어야 하므로 Task 2의 `prisma generate` 선행 필수.

- [ ] **Step 4: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add app/api/commands && git commit -m "feat: /api/commands CRUD"
```

---

## Task 4: 마크다운 렌더러 (`react-markdown`) + 컴포넌트

**Files:**
- Modify: `package.json` (의존성)
- Create: `components/ui/MarkdownText.tsx`

**Interfaces:**
- Produces: `<MarkdownText text={string} />` 기본 export.

- [ ] **Step 1: 의존성 추가**

Run:
```bash
cd /home/server/StoryFit/apps/web && npm install react-markdown remark-gfm
```
Expected: 설치 성공, package.json에 `react-markdown`·`remark-gfm` 추가.

- [ ] **Step 2: 컴포넌트 작성**

`components/ui/MarkdownText.tsx`:

```tsx
'use client'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'

// 커맨드 응답 전용 마크다운 렌더. 원시 HTML은 렌더하지 않음(react-markdown 기본 안전).
export default function MarkdownText({ text }: { text: string }) {
  return (
    <div className="md-body">
      <ReactMarkdown remarkPlugins={[remarkGfm]}>{text}</ReactMarkdown>
    </div>
  )
}
```

- [ ] **Step 3: 최소 스타일**

`app/globals.css` 끝에 추가:

```css
.md-body{ font-size:14px; line-height:1.6; color:var(--ink, #eee); word-break:break-word; }
.md-body h1,.md-body h2,.md-body h3{ margin:0.5em 0 0.3em; font-weight:800; }
.md-body h1{ font-size:1.25em; } .md-body h2{ font-size:1.15em; } .md-body h3{ font-size:1.05em; }
.md-body p{ margin:0.4em 0; } .md-body ul,.md-body ol{ margin:0.4em 0; padding-left:1.3em; }
.md-body li{ margin:0.15em 0; }
.md-body blockquote{ margin:0.5em 0; padding:0.2em 0.8em; border-left:3px solid rgba(255,255,255,0.25); opacity:0.9; }
.md-body hr{ border:none; border-top:1px solid rgba(255,255,255,0.15); margin:0.7em 0; }
.md-body code{ background:rgba(255,255,255,0.08); padding:0.1em 0.35em; border-radius:4px; font-size:0.92em; }
.md-body pre{ background:rgba(255,255,255,0.06); padding:0.7em; border-radius:8px; overflow-x:auto; }
.md-body table{ border-collapse:collapse; margin:0.5em 0; width:100%; }
.md-body th,.md-body td{ border:1px solid rgba(255,255,255,0.15); padding:4px 8px; text-align:left; }
.md-body a{ color:#7cc4ff; text-decoration:underline; }
```

- [ ] **Step 4: 빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: Compiled successfully.

- [ ] **Step 5: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add package.json package-lock.json components/ui/MarkdownText.tsx app/globals.css && git commit -m "feat: MarkdownText(react-markdown) + 스타일"
```

---

## Task 5: 메시지 렌더 분기 (commandName → 마크다운)

**Files:**
- Modify: `app/(main)/conversations/[id]/_components/MessageList.tsx`

**Interfaces:**
- Consumes: `MarkdownText`(Task 4), `Msg`에 `commandName?: string | null`.

- [ ] **Step 1: 타입 확장**

`app/(main)/conversations/[id]/_lib/chatShared.ts`의 `Msg` 인터페이스(메시지 타입)에 필드 추가. `Msg` 정의를 찾아 다음 필드를 추가:

```ts
  commandName?: string | null
```
(이미 `content`·`role` 등이 있는 인터페이스. 같은 블록에 한 줄 추가.)

- [ ] **Step 2: MessageList import + 렌더 분기**

`MessageList.tsx` 상단 import에 추가:

```tsx
import MarkdownText from '@/components/ui/MarkdownText'
```

`MessageList.tsx`에서 assistant 메시지 본문을 `<MessageBlocks text={...} />`로 렌더하는 지점(파일 내 `MessageBlocks`가 메시지 본문에 쓰이는 곳, 약 141·176·416행 부근의 본문 렌더)을 찾아, 해당 본문 렌더를 다음 패턴으로 감싼다 — **`m.commandName`이 있으면 MarkdownText, 없으면 기존 MessageBlocks**:

```tsx
{m.commandName
  ? <MarkdownText text={body} />
  : <MessageBlocks text={body} />}
```

(여기서 `body`는 그 자리에서 MessageBlocks에 넘기던 기존 텍스트 변수. 여러 군데면 "어시스턴트 본문" 렌더 지점에 동일 적용. 스트리밍 중(빈 content) 메시지는 commandName이 있어도 빈 문자열이라 무방.)

- [ ] **Step 3: 빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: Compiled successfully.

- [ ] **Step 4: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add "app/(main)/conversations/[id]/_components/MessageList.tsx" "app/(main)/conversations/[id]/_lib/chatShared.ts" && git commit -m "feat: commandName 메시지는 마크다운으로 렌더"
```

---

## Task 6: 채팅 라우트 커맨드 엔진 확장 (분기 + 지시 주입 + commandName)

**Files:**
- Modify: `app/api/conversations/[id]/chat/route.ts`

**Interfaces:**
- Consumes: `parseCommand`/`isBuiltinCommand`/`builtinFallbackKey`/`BUILTIN_FALLBACK`/`composeCommandDirective`(Task 1), `prisma.userCommand`, `Message.commandName`.

**핵심 설계:** 빌트인이고 데이터가 있으면 기존처럼 **즉시 결정적 응답**(+commandName). 빌트인인데 데이터가 비었거나(폴백) 커스텀이면 **early-return 하지 않고** `commandDirective`/`commandTagName`을 세팅해 **아래 일반 생성 흐름으로 흘려보낸다**. 일반 흐름은 systemPrompt 끝에 directive를 덧붙이고 assistant 메시지에 commandName을 단다.

- [ ] **Step 1: import 추가**

`route.ts` 상단 import 블록에 추가:

```ts
import { parseCommand, isBuiltinCommand, builtinFallbackKey, BUILTIN_FALLBACK, composeCommandDirective } from '@/lib/commands'
```

- [ ] **Step 2: 함수 상단에 커맨드 상태 변수 선언**

`const character = conv.characters[0]?.character` 와 `if (!character) ...` 처리 **직후**, 커맨드 블록 시작 전에 추가:

```ts
  // 커맨드 → AI 생성 경로에서 쓸 값(빌트인 폴백/커스텀). null이면 일반 채팅.
  let commandDirective: string | null = null
  let commandTagName: string | null = null
```

- [ ] **Step 3: 커맨드 블록을 분기형으로 교체**

기존 `if (trimmedInput.startsWith('!')) { ... return NextResponse.json({ messageId: assistantMsg.id }, { status: 200 }) }` 블록 전체(유저 메시지 저장 + cmd 계산 + if/else replyText + assistant 저장 + return)를 아래로 교체:

```ts
  // ── 커맨드 처리 (확장형 커맨드 엔진) ──────────────────────────────────────
  const trimmedInput = content.trim()
  const parsedCmd = parseCommand(trimmedInput)
  if (parsedCmd) {
    const cmd = parsedCmd.name.toLowerCase()
    let replyText: string | null = null // 결정적 즉시 응답. null이면 AI 생성 경로.

    if (isBuiltinCommand(parsedCmd.name)) {
      commandTagName = parsedCmd.name
      // 1) 결정적 빌트인 출력 시도(데이터 있을 때). 빈 데이터면 det=null → 폴백.
      const det = buildBuiltinReply(cmd, conv)
      if (det !== null) {
        replyText = det
      } else {
        // 2) 빈 상태계 커맨드 → AI 폴백
        const fk = builtinFallbackKey(parsedCmd.name)
        if (fk) {
          const extra = parsedCmd.args ? `\n추가 지시: ${parsedCmd.args}` : ''
          commandDirective = `[시스템 커맨드: ${parsedCmd.name}]\n${BUILTIN_FALLBACK[fk]}${extra}\n\n응답은 마크다운 형식으로 작성하라.`
        } else {
          replyText = '*표시할 정보가 없습니다.*'
        }
      }
    } else {
      // 3) 커스텀 커맨드 조회
      const uc = await prisma.userCommand.findUnique({
        where: { userId_name: { userId, name: parsedCmd.name } },
      })
      if (uc) {
        commandTagName = parsedCmd.name
        commandDirective = composeCommandDirective(parsedCmd.name, uc.instruction, parsedCmd.args)
      } else {
        commandTagName = parsedCmd.name
        replyText = `⚠️ **알 수 없는 명령어입니다.**\n사용 가능한 명령어를 보려면 **\`!도움말\`**을 입력해 주세요.`
      }
    }

    // 결정적 즉시 응답 경로
    if (replyText !== null) {
      const prevMsg = conv.messages[conv.messages.length - 1] ?? null
      const userMsg = await prisma.message.create({
        data: { conversationId: params.id, role: 'user', content: trimmedInput, chapter: conv.chapter, isSelected: true, parentId: prevMsg?.id ?? null },
      })
      const assistantMsg = await prisma.message.create({
        data: { conversationId: params.id, role: 'assistant', content: replyText, aiModel: 'system', chapter: conv.chapter, isSelected: true, isStreaming: false, parentId: userMsg.id, commandName: commandTagName },
      })
      return NextResponse.json({ messageId: assistantMsg.id }, { status: 200 })
    }
    // 그 외(commandDirective != null): early-return 없이 아래 일반 생성 흐름으로 진행.
  }
  // ────────────────────────────────────────────────────────────────────────
```

> 주: 기존 `else if (cmd === ...)` 결정적 출력 로직은 아래 `buildBuiltinReply`로 옮긴다(다음 스텝). `getStatsMarkdown`/`getInventoryMarkdown` 등 기존 헬퍼는 그대로 둔다.

- [ ] **Step 4: `buildBuiltinReply` 헬퍼 추가(기존 결정적 로직 이동 + 빈값 null 반환)**

`route.ts` 파일 하단(다른 헬퍼들 근처)에 추가. 기존 if/else의 출력 문자열을 그대로 쓰되, **상태계 커맨드는 데이터가 비었으면 `null` 반환**(폴백 유도). `getStatsMarkdown`/`getInventoryMarkdown`/`도움말` 텍스트는 기존 것 사용:

```ts
function isStatsEmpty(conv: any): boolean {
  return !conv.statsEnabled || !Array.isArray(conv.statsConfig) || conv.statsConfig.length === 0
}
function isInventoryEmpty(conv: any): boolean {
  return !conv.inventoryEnabled || !Array.isArray(conv.inventory) || conv.inventory.length === 0
}
function isSceneEmpty(conv: any): boolean {
  return !conv.statusTimeline || !String(conv.statusTimeline).trim()
}

// 결정적 빌트인 출력. 상태계는 데이터 없으면 null(→ AI 폴백). 도움말/알수없음은 항상 문자열.
function buildBuiltinReply(cmd: string, conv: any): string | null {
  if (cmd === '상태창' || cmd === '정보' || cmd === 'status') {
    if (isStatsEmpty(conv) && isInventoryEmpty(conv) && isSceneEmpty(conv)) return null
    let t = '### 📊 현재 상태창\n\n'
    t += getStatsMarkdown(conv.statsConfig, conv.statsEnabled)
    t += '\n### 🎒 소지품 (인벤토리)\n\n'
    t += getInventoryMarkdown(conv.inventory, conv.inventoryEnabled)
    if (conv.statusTimeline) t += `\n### 🎬 현재 상황\n${conv.statusTimeline}\n`
    return t
  }
  if (cmd === '스탯' || cmd === '능력치' || cmd === 'stats' || cmd === '호감도' || cmd === '관계') {
    if (isStatsEmpty(conv)) return null
    return '### 📊 능력치 및 관계 스탯\n\n' + getStatsMarkdown(conv.statsConfig, conv.statsEnabled)
  }
  if (cmd === '인벤토리' || cmd === '소지품' || cmd === '인벤' || cmd === 'inventory') {
    if (isInventoryEmpty(conv)) return null
    return '### 🎒 소지품 (인벤토리)\n\n' + getInventoryMarkdown(conv.inventory, conv.inventoryEnabled)
  }
  if (cmd === '상황' || cmd === '씬' || cmd === 'scene' || cmd === '타임라인') {
    if (isSceneEmpty(conv)) return null
    return '### 🎬 현재 씬 상황\n\n' + conv.statusTimeline
  }
  if (cmd === '도움말' || cmd === '명령어' || cmd === 'help') {
    return `### ⚙️ StoryFit 시스템 명령어 도움말

대화창에 아래 명령어를 입력하면 AI 비용 없이 즉시 게임 정보를 조회할 수 있습니다.

* **\`!상태창\`** (또는 \`!정보\`) : 스탯, 인벤토리, 현재 상황을 모두 보여줍니다.
* **\`!스탯\`** (또는 \`!호감도\`, \`!관계\`) : 캐릭터와의 관계 및 스탯만 확인합니다.
* **\`!인벤토리\`** (또는 \`!소지품\`) : 가방 속 아이템 목록을 보여줍니다.
* **\`!상황\`** (또는 \`!타임라인\`) : 현재 씬의 시간대, 장소, 의상 등의 상황 요약을 봅니다.
* **\`!도움말\`** : 이 명령어 매뉴얼을 불러옵니다.

> 💡 설정창의 **내 커맨드**에서 나만의 AI 커맨드(예: \`!에타\`)를 만들 수 있습니다.`
  }
  return '⚠️ **알 수 없는 명령어입니다.**'
}
```

- [ ] **Step 5: 일반 생성 흐름에 directive 주입 + commandName 태깅**

`const systemPrompt = buildModeSystemPrompt({ ... })` 호출 **직후**에 한 줄 추가:

```ts
  const finalSystemPrompt = commandDirective ? `${systemPrompt}\n\n${commandDirective}` : systemPrompt
```

그리고 스트리밍 placeholder assistant 메시지 생성부와 `generateAsync` 호출을 `finalSystemPrompt`/`commandName` 사용으로 수정:
- `assistantMsg` 생성 `data`에 `commandName: commandTagName` 추가.
- `generateAsync({ ... systemPrompt, ... })` 호출의 `systemPrompt`를 `finalSystemPrompt`로 교체:

```ts
  const assistantMsg = await prisma.message.create({
    data: {
      conversationId: params.id,
      role: 'assistant',
      content: '',
      aiModel: conv.currentAI,
      chapter: conv.chapter,
      isSelected: true,
      isStreaming: true,
      parentId: userMsg.id,
      commandName: commandTagName,
    },
  })
```
```ts
  generateAsync({
    convId: params.id,
    msgId: assistantMsg.id,
    userId,
    conv,
    character: buildCharParam(character),
    systemPrompt: finalSystemPrompt,
    history,
  }).catch(err => console.error('[chat:async] uncaught error:', err))
```

- [ ] **Step 6: 빌드(타입체크)**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: Compiled successfully.

- [ ] **Step 7: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add "app/api/conversations/[id]/chat/route.ts" && git commit -m "feat: 채팅 커맨드 엔진 — 커스텀/빈값 AI 생성 + commandName"
```

---

## Task 7: 설정창 "내 커맨드" UI(? 가이드) + 자동완성 병합

**Files:**
- Create: `components/ui/CommandGuide.tsx`
- Modify: `app/(main)/conversations/[id]/_components/SidePanel.tsx`
- Modify: `app/(main)/conversations/[id]/page.tsx`, `app/(main)/conversations/[id]/_lib/chatShared.ts`

**Interfaces:**
- Consumes: `/api/commands`(Task 3), `CommandMenu`(기존), `COMMANDS`(기존).

- [ ] **Step 1: 가이드 컴포넌트**

`components/ui/CommandGuide.tsx`:

```tsx
'use client'
import { useState } from 'react'

// 지시문 작성 도움말. 제목 옆 ? 버튼 → 접이식 안내.
export default function CommandGuide() {
  const [open, setOpen] = useState(false)
  return (
    <span style={{ position: 'relative' }}>
      <button type="button" onClick={() => setOpen(o => !o)} aria-label="작성 가이드"
        style={{ marginLeft: 6, width: 16, height: 16, borderRadius: 999, border: '1px solid var(--muted)', background: 'transparent', color: 'var(--muted)', fontSize: 11, lineHeight: 1, cursor: 'pointer' }}>?</button>
      {open && (
        <div style={{ position: 'absolute', zIndex: 30, top: '120%', left: 0, width: 280, background: 'rgba(20,14,26,0.97)', border: '1px solid rgba(255,255,255,0.15)', borderRadius: 10, padding: 12, fontSize: 12, lineHeight: 1.55, color: '#ddd', boxShadow: '0 6px 24px rgba(0,0,0,0.5)' }}>
          <div style={{ fontWeight: 700, marginBottom: 6 }}>지시문 작성 팁</div>
          <ul style={{ margin: 0, paddingLeft: 16 }}>
            <li><b>무엇을·어떤 형식으로</b>: “현재 상황을 확인해 ○○을 ××형식으로 작성하라”</li>
            <li><b>모양 지정</b>: 게시판/채팅 로그/표 등 원하는 형태를 직접 묘사</li>
            <li><b>마크다운</b>: “마크다운으로 작성”이라 쓰면 제목·목록·구분선으로 예쁘게 렌더됩니다</li>
            <li><b>맥락은 자동</b>: 현재 대화·상황을 AI가 알아서 참고. <code>!이름 뒤 텍스트</code>로 추가 지시 가능</li>
          </ul>
          <div style={{ marginTop: 8, padding: 8, background: 'rgba(255,255,255,0.06)', borderRadius: 6 }}>
            예) <b>!에타</b> → “지금까지의 상황을 에브리타임 자유게시판 글처럼 제목 + 본문 + 익명 댓글 몇 개를 마크다운으로 작성하라.”
          </div>
        </div>
      )}
    </span>
  )
}
```

- [ ] **Step 2: SidePanel에 "내 커맨드" 섹션 추가**

`SidePanel.tsx`에 상태·로드·CRUD UI를 추가한다. 파일 상단 import에 `import CommandGuide from '@/components/ui/CommandGuide'`와 `import { api } from '@/lib/api'`(이미 있으면 생략) 추가. 컴포넌트 함수 내부에 상태와 로더를 추가:

```tsx
  const [cmds, setCmds] = useState<{ id: string; name: string; instruction: string; description: string }[]>([])
  const [cmdName, setCmdName] = useState('')
  const [cmdInstr, setCmdInstr] = useState('')
  const [cmdDesc, setCmdDesc] = useState('')
  const [cmdMsg, setCmdMsg] = useState('')
  const loadCmds = () => { api.get('/api/commands').then(setCmds).catch(() => {}) }
  useEffect(() => { loadCmds() }, [])
  const saveCmd = async () => {
    try {
      await api.post('/api/commands', { name: cmdName, instruction: cmdInstr, description: cmdDesc })
      setCmdName(''); setCmdInstr(''); setCmdDesc(''); setCmdMsg('✓ 추가됨'); loadCmds()
    } catch (e: any) { setCmdMsg(`⚠ ${e.message ?? '실패'}`) }
  }
  const delCmd = async (id: string) => { if (!confirm('이 커맨드를 삭제할까요?')) return; await api.delete(`/api/commands/${id}`); loadCmds() }
```

설정 패널의 적절한 섹션 위치(다른 설정 섹션들과 같은 마크업 패턴)로 UI 블록 추가:

```tsx
      <div className="panel-section">
        <div className="panel-section-title">내 커맨드</div>
        <div style={{ fontSize: 11, color: 'var(--muted)', marginBottom: 8 }}>
          나만의 AI 커맨드를 만들면 채팅에서 <code>!이름</code>으로 실행됩니다.
        </div>
        {cmds.map(c => (
          <div key={c.id} style={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between', gap: 8, padding: '6px 0', borderBottom: '1px solid rgba(255,255,255,0.08)' }}>
            <div style={{ minWidth: 0 }}>
              <div style={{ fontWeight: 700, fontSize: 13 }}>!{c.name}</div>
              <div style={{ fontSize: 11, color: 'var(--muted)', overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap' }}>{c.description || c.instruction}</div>
            </div>
            <button onClick={() => delCmd(c.id)} style={{ flexShrink: 0, background: 'none', border: 'none', color: '#ff6b8a', cursor: 'pointer', fontSize: 14 }}>✕</button>
          </div>
        ))}
        <div style={{ display: 'flex', flexDirection: 'column', gap: 6, marginTop: 8 }}>
          <input className="field" placeholder="이름 (예: 에타)" value={cmdName} onChange={e => setCmdName(e.target.value)} />
          <div style={{ display: 'flex', alignItems: 'center', fontSize: 12, fontWeight: 700 }}>지시문 <CommandGuide /></div>
          <textarea className="field" rows={3} placeholder="현재 상황을 확인해 ... 마크다운으로 작성하라" value={cmdInstr} onChange={e => setCmdInstr(e.target.value)} style={{ resize: 'vertical', fontSize: 12 }} />
          <input className="field" placeholder="설명(선택, 자동완성에 표시)" value={cmdDesc} onChange={e => setCmdDesc(e.target.value)} />
          <button className="btn" onClick={saveCmd} disabled={!cmdName.trim() || !cmdInstr.trim()}>+ 커맨드 추가</button>
          {cmdMsg && <div style={{ fontSize: 11, color: cmdMsg.startsWith('✓') ? '#4ade80' : '#ff6b8a' }}>{cmdMsg}</div>}
        </div>
      </div>
```

(클래스명 `panel-section`/`panel-section-title`/`field`/`btn`은 SidePanel 기존 패턴에 맞춤. 실제 파일의 섹션 클래스명을 확인해 일치시킨다. `useState`/`useEffect`가 이미 import돼 있는지 확인.)

- [ ] **Step 3: 자동완성에 유저 커맨드 병합**

`app/(main)/conversations/[id]/page.tsx`에서 유저 커맨드를 불러와 `COMMANDS`와 합쳐 자동완성에 쓴다. 상단에 `import { api } from '@/lib/api'`(있으면 생략). 컴포넌트 내부에 추가:

`COMMANDS`는 `{ name: string; desc: string }[]` 형태이고, 기존 필터는
`const filteredCommands = COMMANDS.filter(cmd => cmd.name.toLowerCase().startsWith(commandQuery.toLowerCase()))` 이다.
유저 커맨드도 같은 `{ name: '!에타', desc }` 형태로 맞춰 병합한다. 컴포넌트 내부에 추가:

```tsx
  const [userCmds, setUserCmds] = useState<{ name: string; desc: string }[]>([])
  useEffect(() => {
    api.get('/api/commands')
      .then((cs: { name: string; description: string }[]) =>
        setUserCmds(cs.map(c => ({ name: `!${c.name}`, desc: c.description || '내 커맨드' }))))
      .catch(() => {})
  }, [])
```

그리고 기존 `const filteredCommands = COMMANDS.filter(...)` 줄을 **정확히** 아래로 교체(대상 배열만 병합):

```tsx
  const filteredCommands = [...COMMANDS, ...userCmds].filter(cmd =>
    cmd.name.toLowerCase().startsWith(commandQuery.toLowerCase())
  )
```
(`CommandMenu`/`selectCommand`는 `{name, desc}`·`name` 문자열을 그대로 받으므로 추가 변경 불필요.)

- [ ] **Step 4: 빌드**

Run: `cd /home/server/StoryFit/apps/web && npm run build`
Expected: Compiled successfully.

- [ ] **Step 5: 수동 검증(배포/로그인 서버 필요 — 이월)**

설정창에서 `!에타` 커맨드 생성 → 채팅에서 `!에타` 입력 시 마크다운 AI 응답 생성, 자동완성에 `!에타` 노출, 빈 상태에서 `!상태창`이 AI 마크다운 상태창 생성. (로그인 서버 필요 → 사용자 검증)

- [ ] **Step 6: 커밋**

```bash
cd /home/server/StoryFit/apps/web && git add components/ui/CommandGuide.tsx "app/(main)/conversations/[id]/_components/SidePanel.tsx" "app/(main)/conversations/[id]/page.tsx" "app/(main)/conversations/[id]/_lib/chatShared.ts" && git commit -m "feat: 설정창 내 커맨드 CRUD(? 가이드) + 자동완성 병합"
```

---

## Task 8: 가이드 페이지 동기화

**Files:**
- Modify: `app/(main)/guide/page.tsx`

- [ ] **Step 1: FEATURE_SECTIONS 항목 추가**

`guide/page.tsx`의 `FEATURE_SECTIONS`에 커맨드 관련 항목을 추가(기존 항목 형식에 맞춤). 내용 요지:
- `!상태창`·`!스탯`·`!인벤토리`·`!상황`·`!도움말` 기본 커맨드 + 값 없으면 AI가 상태창 생성.
- 설정창 "내 커맨드"에서 `!에타` 같은 나만의 AI 커맨드 생성·재사용, 응답은 마크다운 렌더.

(기존 `FEATURE_SECTIONS` 한 항목의 구조를 그대로 따라 작성.)

- [ ] **Step 2: 빌드 + 커밋**

Run: `cd /home/server/StoryFit/apps/web && npm run build` → Compiled successfully.
```bash
cd /home/server/StoryFit/apps/web && git add "app/(main)/guide/page.tsx" && git commit -m "docs: 가이드에 커스텀 커맨드/상태창 AI 폴백 안내 추가"
```

---

## 실행 순서/의존성

- Task 1 → 2 → 3(2 필요) → 4 → 5(4 필요) → 6(1·2·5 필요) → 7(3 필요) → 8.
- Task 1만 vitest TDD. 나머지는 빌드(타입체크)+수동. 라이브 동작(커맨드 생성/실행/마크다운/폴백)은 로그인 서버에서 사용자 검증.
- 배포: apps/web main 머지 → 부모 master 포인터 → 서버 `git pull && git submodule update --remote && docker compose up --build -d` (마이그레이션이 적용되도록 컨테이너 기동 시 `prisma migrate deploy` 포함 여부 확인 — 미포함이면 서버에서 수동 적용).
