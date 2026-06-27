# 프롬프트 3종 개선 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 빠른전개 채팅 토글 + 응답 길이 min/max 설정 + 멀티 모드 선택지 출력을 추가한다.

**Architecture:** 코드(스키마·systemPrompt·UI)는 git, 규칙 텍스트(modeRules·multiStory_closing)는 라이브 DB(GlobalConfig) SQL로 분리 적용. 순수 프롬프트 로직은 Vitest로 검증.

**Tech Stack:** Next.js 14, React, Prisma(db push), Vitest, PostgreSQL(docker `storyfit-db-1`).

설계 문서: `context/2026-06-27-prompt-improvements-design.md`

## Global Constraints

- 시스템 프롬프트 조립 순서 불변(CLAUDE.md): 빠른전개 블록만 **modeRules 직후(정적 프리픽스)**에 추가, 그 외 순서 불변.
- 빠른전개 기본값 = off(`@default(false)`). 길이 기본 = 비움(미적용).
- 멀티 선택지: 1·2=페르소나 행동/대사, 3·4=캐릭터 대사/행동(이번 응답 등장 캐릭터 기본, 흐름상 새 캐릭터 가능). 미등장 제외 제약 없음.
- DB SQL은 **라이브 반영** — 수정 전 현재 값을 파일로 백업, 수정 후 SELECT로 검증.
- 작업 디렉터리 `apps/web`. 코드 커밋은 apps/web. DB 변경은 git과 무관(별도). 배포는 전체 완료 후 사용자 확인.
- 검증: `npx tsc --noEmit` exit 0 + `npx vitest run`.

---

### Task 1: 빠른전개 — 스키마 + 프롬프트 블록 + 배선 (백엔드)

**Files:**
- Modify: `prisma/schema.prisma` (model Conversation)
- Modify: `lib/systemPrompt.ts` (FAST_PACE_BLOCK 상수 + 두 빌더 param/주입)
- Modify: `lib/chatPipeline.ts` (`buildModeSystemPrompt`)
- Modify: `app/api/conversations/[id]/chat/route.ts`, `.../regenerate/route.ts`, `.../continue/route.ts`
- Modify: `app/api/conversations/[id]/route.ts` (PATCH 허용목록)
- Modify: `app/api/conversations/route.ts` (POST 수용 — 선택)
- Test: `lib/systemPrompt.test.ts`

**Interfaces:**
- Produces: `buildStorySystemPrompt`/`buildMultiStorySystemPrompt`가 `fastPace?: boolean` 수용. `Conversation.fastPaceEnabled: boolean`(default false).

- [ ] **Step 1: 실패 테스트** — `lib/systemPrompt.test.ts` 끝에 추가

```ts
describe('빠른전개(fastPace)', () => {
  const character = { name: '지영', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' } as any
  it('fastPace=true면 빠른 전개 블록이 포함된다', () => {
    const out = buildStorySystemPrompt({ character, fastPace: true })
    expect(out).toContain('[전개 속도 — 다른 모든 속도 지시보다 우선]')
  })
  it('fastPace 미지정이면 빠른 전개 블록이 없다', () => {
    const out = buildStorySystemPrompt({ character })
    expect(out).not.toContain('[전개 속도 — 다른 모든 속도 지시보다 우선]')
  })
})
```

- [ ] **Step 2: 실패 확인** — Run: `npx vitest run lib/systemPrompt.test.ts` → FAIL(블록 없음).

- [ ] **Step 3: 구현**
  1. `lib/systemPrompt.ts` 상단(함수들 위)에 상수 추가:

```ts
const FAST_PACE_BLOCK = `[전개 속도 — 다른 모든 속도 지시보다 우선]
- 이번 대화는 빠르게 전개한다: 시간·장소를 과감히 건너뛰고, 한 응답에서 사건을 여러 단계 진행시켜 상황을 크게 움직인다.
- 군더더기 묘사·뜸들이는 서술을 줄이고 핵심 전개에 집중한다.
- 이 지시는 [공통 문체]의 "느리게 서술"·"결론을 서두르지 않는다"보다 우선한다.`
```

  2. `BuildSystemPromptParams`와 `MultiStoryPromptParams`에 `fastPace?: boolean` 추가. 두 함수 구조분해에 `fastPace = false` 추가.
  3. 두 함수 모두 `if (modeRules?.trim()) parts.push(...)` **직후**에:

```ts
  if (fastPace) parts.push(FAST_PACE_BLOCK)
```

  4. `prisma/schema.prisma` model Conversation의 `personaFlipPlaceholders` 아래:

```prisma
  fastPaceEnabled         Boolean              @default(false)
```

  5. `npx prisma db push && npx prisma generate`.
  6. `lib/chatPipeline.ts` `buildModeSystemPrompt`: 구조분해·타입에 `fastPace` 추가, 두 return에 전달(`flipPersonaPlaceholders` 옆):

```ts
  if (mode === 'multiStory') return buildMultiStorySystemPrompt({ ...base, characters, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders, fastPace })
  return buildStorySystemPrompt({ ...base, character, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders, fastPace })
```

  7. chat·regenerate·continue 3개 라우트의 `buildModeSystemPrompt({...})`에서 `flipPersonaPlaceholders:` 줄 아래에:

```ts
    fastPace: conv.fastPaceEnabled ?? false,
```

  8. `app/api/conversations/[id]/route.ts`의 PATCH `allowed` 배열에 `'fastPaceEnabled'` 추가.
  9. `app/api/conversations/route.ts` POST create data에(선택, 기본 false라 없어도 무방하지만 명시):

```ts
      fastPaceEnabled: typeof body.fastPaceEnabled === 'boolean' ? body.fastPaceEnabled : false,
```

- [ ] **Step 4: 통과 확인** — Run: `npx vitest run lib/systemPrompt.test.ts && npx tsc --noEmit` → PASS, exit 0.

- [ ] **Step 5: 커밋**

```bash
git add prisma/schema.prisma lib/systemPrompt.ts lib/systemPrompt.test.ts lib/chatPipeline.ts "app/api/conversations/[id]/chat/route.ts" "app/api/conversations/[id]/regenerate/route.ts" "app/api/conversations/[id]/continue/route.ts" "app/api/conversations/[id]/route.ts" app/api/conversations/route.ts
git commit -m "feat: 빠른전개 토글(fastPaceEnabled) — 프롬프트 블록 + 배선"
```

---

### Task 2: 빠른전개 — SidePanel 토글 UI

**Files:**
- Modify: `app/(main)/conversations/[id]/_components/SidePanel.tsx` (기존 토글들 근처 ~330-360)

**Interfaces:**
- Consumes: Task 1의 PATCH `fastPaceEnabled`.

- [ ] **Step 1: 토글 추가** — `enrichInputMode` 토글 블록을 복제해 바로 아래에 추가(라벨만 변경):

```tsx
        <label className="row" style={{ display: 'flex', alignItems: 'center', gap: 8, padding: '6px 0' }}>
          <span className="tiny" style={{ flex: 1 }}>빠른 전개</span>
          <input
            type="checkbox"
            checked={(conv as any).fastPaceEnabled ?? false}
            onChange={async e => {
              const checked = e.target.checked
              await api.patch(`/api/conversations/${convId}`, { fastPaceEnabled: checked })
              setConv(c => c ? ({ ...c, fastPaceEnabled: checked } as any) : c)
            }}
          />
          <span className="tiny">{(conv as any).fastPaceEnabled ? 'ON' : 'OFF'}</span>
        </label>
```

> 구현 시 SidePanel의 실제 토글 마크업(autoChapterEnabled/enrichInputMode/personaAutoMode, line ~305-360)을 읽고 동일한 클래스·구조로 맞춘다.

- [ ] **Step 2: 타입체크** — Run: `npx tsc --noEmit` → exit 0.

- [ ] **Step 3: 커밋**

```bash
git add "app/(main)/conversations/[id]/_components/SidePanel.tsx"
git commit -m "feat: SidePanel 빠른전개 토글"
```

---

### Task 3: 응답 길이 — 타입 + buildStyleSection + 테스트

**Files:**
- Modify: `types/index.ts` (StyleConfig.length)
- Modify: `lib/systemPrompt.ts` (buildStyleSection length 처리)
- Test: `lib/systemPrompt.test.ts`

**Interfaces:**
- Produces: `StyleConfig.length: { min?: number; max?: number } | null`. buildStyleSection이 객체 length를 "응답 길이: …자"로 출력.

- [ ] **Step 1: 실패 테스트** — `lib/systemPrompt.test.ts`에 추가

```ts
describe('응답 길이 min/max', () => {
  const character = { name: '지영', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' } as any
  it('min·max 모두 있으면 범위로 출력', () => {
    const out = buildStorySystemPrompt({ character, styleConfig: { length: { min: 300, max: 600 } } as any })
    expect(out).toContain('응답 길이: 300~600자')
  })
  it('레거시 문자열 length는 무시(출력 없음)', () => {
    const out = buildStorySystemPrompt({ character, styleConfig: { length: '짧게' } as any })
    expect(out).not.toContain('응답 길이')
  })
})
```

- [ ] **Step 2: 실패 확인** — Run: `npx vitest run lib/systemPrompt.test.ts` → FAIL.

- [ ] **Step 3: 구현**
  1. `types/index.ts`: `length?: '짧게' | '보통' | '길게' | null` → `length?: { min?: number; max?: number } | null`.
  2. `lib/systemPrompt.ts` buildStyleSection의 length 줄 교체:

```ts
  if (s.length && typeof s.length === 'object') {
    const { min, max } = s.length
    if (min && max)    lines.push(`- 응답 길이: ${min}~${max}자`)
    else if (min)      lines.push(`- 응답 길이: 최소 ${min}자 이상`)
    else if (max)      lines.push(`- 응답 길이: 최대 ${max}자 이내`)
  }
```

- [ ] **Step 4: 통과 확인** — Run: `npx vitest run lib/systemPrompt.test.ts && npx tsc --noEmit`.
  ⚠️ tsc가 length를 문자열로 쓰던 UI 파일(StyleSection·SidePanel)에서 에러를 낼 수 있음 — Task 4에서 해소. 본 태스크는 `npx tsc --noEmit 2>&1 | grep -E "systemPrompt|types/index" || echo "core OK"`로 코어만 확인하고 커밋.

- [ ] **Step 5: 커밋**

```bash
git add types/index.ts lib/systemPrompt.ts lib/systemPrompt.test.ts
git commit -m "feat: 응답 길이 min/max 자수 타입 + buildStyleSection"
```

---

### Task 4: 응답 길이 — UI(StyleSection + SidePanel) min/max 입력

**Files:**
- Modify: `app/(main)/conversations/new/_components/StyleSection.tsx` (length 칩 → 숫자 입력)
- Modify: `app/(main)/conversations/[id]/_components/SidePanel.tsx` (length 칩 → 숫자 입력)

**Interfaces:**
- Consumes: Task 3의 `StyleConfig.length` 객체 타입.

- [ ] **Step 1: StyleSection.tsx** — 옵션 배열에서 `{ key: 'length', ... }` 항목 제거하고, 칩 맵 아래에 최소/최대 입력 행 추가:

```tsx
        <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
          <span style={{ fontSize: 11, fontWeight: 600, width: 60, flexShrink: 0 }}>응답 길이</span>
          <input type="number" min={0} placeholder="최소" style={{ width: 70 }}
            value={style?.length?.min ?? ''}
            onChange={e => onChange({ ...style, length: { ...(style?.length ?? {}), min: e.target.value ? Number(e.target.value) : undefined } })} />
          <span className="muted">~</span>
          <input type="number" min={0} placeholder="최대" style={{ width: 70 }}
            value={style?.length?.max ?? ''}
            onChange={e => onChange({ ...style, length: { ...(style?.length ?? {}), max: e.target.value ? Number(e.target.value) : undefined } })} />
          <span className="muted" style={{ fontSize: 11 }}>자</span>
        </div>
```

> 구현 시 StyleSection의 실제 prop명(현재 styleConfig 상태/세터)을 확인해 `style`/`onChange`를 맞춘다.

- [ ] **Step 2: SidePanel.tsx** — 옵션 배열(line ~399-404)에서 `length` 제거하고, 동일한 최소/최대 입력 행 추가. 값 세팅은 기존 `handleStyleConfig`가 string 전용이므로 length 전용 핸들러를 추가:

```tsx
  const setLength = (patch: { min?: number; max?: number }) => {
    const cur = (conv?.styleConfig as any)?.length ?? {}
    const next = { ...(conv?.styleConfig ?? {}), length: { ...cur, ...patch } }
    setConv(c => c ? { ...c, styleConfig: next } : c)
    api.patch(`/api/conversations/${convId}`, { styleConfig: next }).catch(() => setToast('스타일 저장에 실패했습니다'))
  }
```

입력 행:

```tsx
        <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
          <span style={{ fontSize: 11, fontWeight: 600, width: 60, flexShrink: 0 }}>응답 길이</span>
          <input type="number" min={0} placeholder="최소" style={{ width: 64 }}
            value={(conv.styleConfig as any)?.length?.min ?? ''}
            onChange={e => setLength({ min: e.target.value ? Number(e.target.value) : undefined })} />
          <span className="muted">~</span>
          <input type="number" min={0} placeholder="최대" style={{ width: 64 }}
            value={(conv.styleConfig as any)?.length?.max ?? ''}
            onChange={e => setLength({ max: e.target.value ? Number(e.target.value) : undefined })} />
          <span className="muted" style={{ fontSize: 11 }}>자</span>
        </div>
```

- [ ] **Step 3: 전체 타입체크** — Run: `npx tsc --noEmit` → exit 0(length 문자열 사용처가 모두 정리됨).

- [ ] **Step 4: 커밋**

```bash
git add "app/(main)/conversations/new/_components/StyleSection.tsx" "app/(main)/conversations/[id]/_components/SidePanel.tsx"
git commit -m "feat: 응답 길이 min/max 입력 UI(StyleSection·SidePanel)"
```

---

### Task 5: 멀티 base — 선택지 금지 줄 제거

**Files:**
- Modify: `lib/systemPrompt.ts` (멀티 baseRules)
- Test: `lib/systemPrompt.test.ts`

- [ ] **Step 1: 실패 테스트** — 추가

```ts
describe('멀티 base 선택지 허용', () => {
  const chars = [{ name: 'A', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' }, { name: 'B', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' }] as any
  it('멀티 base에 선택지 금지 문구가 없다', () => {
    const out = buildMultiStorySystemPrompt({ characters: chars })
    expect(out).not.toContain('Do NOT offer choices')
  })
})
```

- [ ] **Step 2: 실패 확인** — Run: `npx vitest run lib/systemPrompt.test.ts` → FAIL.

- [ ] **Step 3: 구현** — 멀티 `baseRules`에서 아래 교체:

old:
```
- At least one character must take direct action or deliver dialogue, then naturally advance the scene.
- Do NOT offer choices, numbered options, or host-like questions. Never append a "---" divider followed by a list of options. End with the scene itself.
- ${personaRule}
```
new:
```
- At least one character must take direct action or deliver dialogue, then naturally advance the scene.
- ${personaRule}
```

- [ ] **Step 4: 통과 확인** — Run: `npx vitest run lib/systemPrompt.test.ts && npx tsc --noEmit`.

- [ ] **Step 5: 커밋**

```bash
git add lib/systemPrompt.ts lib/systemPrompt.test.ts
git commit -m "feat(multi): base의 선택지 금지 규칙 제거(멀티 선택지 허용)"
```

---

### Task 6: DB GlobalConfig SQL 수정 (라이브)

> ⚠️ git 아님 — 라이브 DB 변경. 백업 → 수정 → 검증.

**대상 키:** `story_rules`, `multiStory_rules`(600~800자 줄 제거), `multiStory_closing`(선택지 규칙으로 교체).

- [ ] **Step 1: 현재 값 백업**

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT key, value FROM \"GlobalConfig\" WHERE key IN ('story_rules','multiStory_rules','multiStory_closing');" > /home/server/StoryFit/apps/web/.superpowers/sdd/globalconfig-backup-$(date +%s).txt
echo "backup saved"
```

- [ ] **Step 2: 600~800자 줄 제거(story_rules·multiStory_rules)**

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -c "UPDATE \"GlobalConfig\" SET value = replace(value, E'\n  - 본문은 600~800자로 서술한다', '') WHERE key IN ('story_rules','multiStory_rules');"
```

- [ ] **Step 3: multiStory_closing 교체**

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -v ON_ERROR_STOP=1 <<'SQL'
UPDATE "GlobalConfig" SET value =
'[Top Priority]
- Always respond in Korean only
- Never output analysis, planning, or meta-commentary
- Begin directly with scene, dialogue, or action
- In the body, never write the persona(user)''s words, actions, emotions, or decisions
- Characters perform only their own words and actions, then naturally advance the scene
- Write richly with narration, action, and dialogue
- REQUIRED: Do not repeat dialogue, actions, or narration already present in the previous exchange
- At the end, place a "---" divider and present 4 numbered choices:
  - Choices 1-2: the persona(user)''s next action or dialogue
  - Choices 3-4: a character''s next dialogue or action (prefer characters present in this response; a new character may appear if the flow naturally calls for one)'
WHERE key = 'multiStory_closing';
SQL
```

- [ ] **Step 4: 검증**

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT (value LIKE '%600~800%') AS has_len_story FROM \"GlobalConfig\" WHERE key='story_rules';"
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT (value LIKE '%600~800%') AS has_len_multi FROM \"GlobalConfig\" WHERE key='multiStory_rules';"
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT (value LIKE '%4 numbered choices%') AS has_choices FROM \"GlobalConfig\" WHERE key='multiStory_closing';"
```
Expected: `f`, `f`, `t`.

---

### Task 7: 가이드 동기화 + 최종 검증

**Files:**
- Modify: `app/(main)/guide/page.tsx`

- [ ] **Step 1: 가이드 항목 추가** — `🎛 고급 AI 파라미터` 또는 `⚙️ 대화 설정` 섹션에:

```ts
      { emoji: '⏩', label: '빠른 전개', desc: '채팅방 사이드패널에서 빠른 전개를 켜면 시간·장소를 과감히 건너뛰고 사건을 여러 단계 진행시켜 이야기를 빠르게 전개합니다(기본 꺼짐).' },
      { emoji: '📏', label: '응답 길이(최소·최대)', desc: '스타일 설정의 응답 길이를 최소·최대 글자 수로 지정할 수 있습니다(비워두면 제한 없음).' },
```

- [ ] **Step 2: 전체 검증** — Run: `npx tsc --noEmit && npx vitest run` → exit 0, 전부 PASS.

- [ ] **Step 3: 커밋**

```bash
git add "app/(main)/guide/page.tsx"
git commit -m "docs: 빠른전개·응답길이 기능 가이드 동기화"
```

---

## 배포 (전체 완료 후, 사용자 확인 하에)

```bash
cd apps/web && git push origin HEAD:main
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (빠른전개·길이·멀티선택지)" && git push origin master
```
DB(Task 6)는 라이브에 이미 반영됨(별도 배포 불필요). 코드 배포 후 `docker compose up --build -d`.

## Self-Review 메모

- 스펙 커버리지: 빠른전개(T1·T2), 길이 min/max(T3·T4 + DB 600자제거 T6), 멀티선택지(T5 base + T6 closing), 가이드(T7) — 전부 매핑.
- 타입 일관성: `fastPace`/`fastPaceEnabled`, `StyleConfig.length:{min?,max?}` 전 태스크 일관.
- ⚠️ T4 구현 시 StyleSection/SidePanel의 실제 prop·핸들러명을 읽고 맞출 것. T1·T2의 SidePanel 토글 마크업은 기존 토글과 동일 클래스로.
- DB SQL(T6)은 라이브 — 백업 필수, 검증 SELECT 확인.
