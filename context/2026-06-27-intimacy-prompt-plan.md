# 성애/친밀 장면 프롬프트 개선 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 친밀 장면 프롬프트를 관계 맥락(톤)·전희 디테일(품질)·합의 게이팅 토글로 개선한다.

**Architecture:** 품질·톤 규칙은 `modeRules`(DB) 편집, 진입 게이팅은 코드 상수 `INTIMACY_GATING_BLOCK`를 `adultGatingEnabled`일 때만 주입(fastPace 패턴). 순수 프롬프트 로직은 Vitest 검증.

**Tech Stack:** Next.js 14, React, Prisma, Vitest, PostgreSQL(docker `storyfit-db-1`).

설계: `context/2026-06-27-intimacy-prompt-design.md`

## Global Constraints

- 하드리밋(미성년·실제 비합의·CSAM)은 우리 프롬프트에 없고 모델이 강제. 본 작업은 이를 건드리지 않으며, 어떤 토글도 모델 안전 우회 문구를 추가하지 않는다(OFF=게이팅 블록 미주입).
- 게이팅 토글 기본값 = ON(`@default(true)`). 관계 단계는 진입 게이트가 아니라 톤 참고(AI 자동 판단).
- 시스템 프롬프트 조립 순서 불변: 게이팅 블록만 modeRules 직후(fastPace와 같은 정적 위치).
- DB SQL은 라이브 — 백업 후 적용, SELECT로 검증.
- 작업 디렉터리 `apps/web`. 코드 커밋은 apps/web. 배포는 누적 변경과 함께 사용자 확인 하에.
- 검증: `npx tsc --noEmit` exit 0 + `npx vitest run`.

---

### Task 1: 합의 게이팅 — 스키마 + 코드 블록 + 배선

**Files:**
- Modify: `prisma/schema.prisma`, `lib/systemPrompt.ts`, `lib/chatPipeline.ts`, `app/api/conversations/[id]/{chat,regenerate,continue}/route.ts`, `app/api/conversations/[id]/route.ts`, `app/api/conversations/route.ts`
- Test: `lib/systemPrompt.test.ts`

**Interfaces:**
- Produces: `buildStorySystemPrompt`/`buildMultiStorySystemPrompt`가 `adultGating?: boolean`(기본 true) 수용. `Conversation.adultGatingEnabled: boolean`(default true).

- [ ] **Step 1: 실패 테스트** — `lib/systemPrompt.test.ts` 끝에 추가

```ts
describe('합의 게이팅(adultGating)', () => {
  const character = { name: '지영', kind: 'custom', safetyLevel: 'standard', defaultAI: 'gemini' } as any
  it('adultGating 미지정(기본 true)이면 게이팅 블록 포함', () => {
    const out = buildStorySystemPrompt({ character })
    expect(out).toContain('[성애 진입 — 합의·맥락 전제]')
  })
  it('adultGating=false면 게이팅 블록 없음', () => {
    const out = buildStorySystemPrompt({ character, adultGating: false })
    expect(out).not.toContain('[성애 진입 — 합의·맥락 전제]')
  })
})
```

- [ ] **Step 2: 실패 확인** — Run: `npx vitest run lib/systemPrompt.test.ts` → FAIL.

- [ ] **Step 3: 구현**
  1. `lib/systemPrompt.ts` 상단의 `FAST_PACE_BLOCK` 아래에 상수 추가:

```ts
const INTIMACY_GATING_BLOCK = `[성애 진입 — 합의·맥락 전제]
- 성애 장면은 서사가 자연스럽게 그 방향으로 흐르고 두 인물이 명확한 욕망을 주고받았을 때, 또는 사용자가 그 방향으로 명확히 이끌 때 시작한다.
- 한쪽이 거부·주저하는 신호를 보이면 그 의사를 존중해 진행을 멈추거나 속도를 늦춘다.`
```

  2. `BuildSystemPromptParams`·`MultiStoryPromptParams`에 `adultGating?: boolean` 추가. 두 함수 구조분해에 `adultGating = true` 추가.
  3. 두 함수에서 `if (fastPace) parts.push(FAST_PACE_BLOCK)` 바로 아래에:

```ts
  if (adultGating) parts.push(INTIMACY_GATING_BLOCK)
```

  4. 스키마: `prisma/schema.prisma` model Conversation의 `fastPaceEnabled` 아래:

```prisma
  adultGatingEnabled      Boolean              @default(true)
```

  5. DB 컬럼 추가(호스트에서 `db:5432` 안 닿으므로 docker exec로 직접 ALTER) + 클라이언트 재생성:

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -c "ALTER TABLE \"Conversation\" ADD COLUMN IF NOT EXISTS \"adultGatingEnabled\" BOOLEAN NOT NULL DEFAULT true;"
npx prisma generate
```

  6. `lib/chatPipeline.ts` `buildModeSystemPrompt`: 구조분해·타입에 `adultGating` 추가, 두 return에 전달:

```ts
  if (mode === 'multiStory') return buildMultiStorySystemPrompt({ ...base, characters, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders, fastPace, adultGating })
  return buildStorySystemPrompt({ ...base, character, statsConfig, inventory, allowPersonaDialogue, flipPersonaPlaceholders, fastPace, adultGating })
```

  7. chat·regenerate·continue 3개 라우트의 `buildModeSystemPrompt({...})`에서 `fastPace:` 줄 아래에:

```ts
    adultGating: conv.adultGatingEnabled ?? true,
```

  8. `app/api/conversations/[id]/route.ts` PATCH `allowed` 배열에 `'adultGatingEnabled'` 추가.
  9. `app/api/conversations/route.ts` POST create data에:

```ts
      adultGatingEnabled: typeof body.adultGatingEnabled === 'boolean' ? body.adultGatingEnabled : true,
```

- [ ] **Step 4: 통과 확인** — Run: `npx vitest run lib/systemPrompt.test.ts && npx tsc --noEmit` → PASS, exit 0.

- [ ] **Step 5: 커밋**

```bash
git add prisma/schema.prisma lib/systemPrompt.ts lib/systemPrompt.test.ts lib/chatPipeline.ts "app/api/conversations/[id]/chat/route.ts" "app/api/conversations/[id]/regenerate/route.ts" "app/api/conversations/[id]/continue/route.ts" "app/api/conversations/[id]/route.ts" app/api/conversations/route.ts
git commit -m "feat: 성인 합의 게이팅 토글(adultGatingEnabled) — 게이팅 블록 코드 분리 + 배선"
```

---

### Task 2: SidePanel 게이팅 토글 UI

**Files:**
- Modify: `app/(main)/conversations/[id]/_components/SidePanel.tsx` (빠른전개 토글 섹션 아래)

- [ ] **Step 1: 토글 추가** — `⏩ 빠른 전개` 토글 `</div>` 섹션 바로 아래에(빠른전개와 동일 구조로):

```tsx
      <div className="side-section" hidden={tab !== 'ai'}>
        <div className="spread" style={{ alignItems: 'center' }}>
          <div className="label" style={{ marginBottom: 0 }}>🔞 성인 합의 게이팅</div>
          <label className="hstack" style={{ gap: 6, cursor: 'pointer' }}>
            <input
              type="checkbox"
              checked={(conv as any).adultGatingEnabled ?? true}
              onChange={async e => {
                const checked = e.target.checked
                try {
                  await api.patch(`/api/conversations/${convId}`, { adultGatingEnabled: checked })
                  setConv(c => c ? ({ ...c, adultGatingEnabled: checked } as any) : c)
                } catch { setToast('설정 저장에 실패했습니다') }
              }}
            />
            <span className="tiny">{(conv as any).adultGatingEnabled ?? true ? 'ON' : 'OFF'}</span>
          </label>
        </div>
        <div className="tiny muted" style={{ marginTop: 4 }}>ON이면 성애 장면 진입에 합의·맥락 전제를 둡니다. OFF면 우리 측 진입 게이팅을 해제합니다(모델 자체 안전선은 유지).</div>
      </div>
```

> 구현 시 빠른전개 토글(`fastPaceEnabled`) 마크업을 읽고 동일 클래스로 맞춘다. 타입 단언은 `chatShared.ts`의 `Conv`에 필드가 없으면 `(conv as any)`로.

- [ ] **Step 2: 타입체크** — Run: `npx tsc --noEmit` → exit 0.

- [ ] **Step 3: 커밋**

```bash
git add "app/(main)/conversations/[id]/_components/SidePanel.tsx"
git commit -m "feat: SidePanel 성인 합의 게이팅 토글"
```

---

### Task 3: DB modeRules 편집 (라이브) — 게이팅 줄 제거 + 관계맥락·전희 블록 추가

> ⚠️ git 아님 — 라이브 DB. 백업 → 적용 → 검증. story_rules·multiStory_rules 둘 다.

- [ ] **Step 1: 백업**

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT key, value FROM \"GlobalConfig\" WHERE key IN ('story_rules','multiStory_rules');" > /home/server/StoryFit/apps/web/.superpowers/sdd/modeRules-backup-$(date +%s).txt
echo "backup saved"
```

- [ ] **Step 2: 게이팅 두 줄 제거 + 신규 블록 삽입** — 한 UPDATE로(둘 다):

```bash
docker exec -i storyfit-db-1 psql -U storyfit -d storyfit -v ON_ERROR_STOP=1 <<'SQL'
UPDATE "GlobalConfig"
SET value = replace(
  regexp_replace(
    replace(value, E'\n  - 로맨스의 긴장과 성애는 다르다. 로맨스 단계에서는 노골적 신체 묘사를 사용하지 않는다', ''),
    E'\n  - 성애 장면은 서사가.*?시작한다', '', 'gs'
  ),
  '  [성애 장면 전용 — 위 조건이 충족된 경우에만 적용]',
  E'  [관계 맥락 — 톤을 정하는 참고(진입을 막는 조건이 아님)]\n  - 두 인물의 현재 관계 맥락을 판단해(답변엔 미포함) 친밀 장면의 심리·태도·온도를 거기에 맞춘다. 예: 낯선 사이·원나잇 → 낯섦·긴장·즉흥적 욕망 / 호감 → 설렘·탐색·서툶 / 연인·부부 → 애정·익숙함·신뢰\n  - 관계 단계는 성애 진입을 막는 게이트가 아니다. 원나잇, 호감 단계의 잠자리, 이후 연애로의 발전 등 어떤 전개도 가능하다.\n  - 같은 행위라도 관계 맥락에 따라 감정의 결과 대사 톤을 다르게 표현한다.\n\n  [친밀 장면 속도·전희 — 절정보다 우선]\n  - 절정으로 곧장 가지 않는다. 분위기 조성 → 접촉·애무(전희) → 점진적 고조 순으로 천천히 진행한다.\n  - 전희를 한 덩어리로 뭉뚱그리지 않는다. 손길·입술이 닿는 부위를 하나씩 따라가며(목덜미·쇄골·가슴·허리·허벅지 등) 각 부위의 애무를 차례로 구체적으로 묘사한다.\n  - 한 부위에 충분히 머문 뒤 다음 부위로 옮겨가고, 매 동작마다 감각(촉감·온도·압력·소리)과 상대의 반응(떨림·호흡·목소리)을 짝지어 서술한다.\n  - 응답 분량의 대부분을 전희·애무에 쓴다. 절정은 장면의 전희가 충분히 무르익고 사용자가 그 방향으로 명확히 이끌 때 다다른다.\n  - 다음 수위로 넘어갈 때마다 상대의 반응을 살피며 강도를 한 단계씩 올린다.\n  - 친밀 장면에서는 이 전희·속도 규칙이 빠른 전개 지시보다 우선한다.\n\n  [성애 장면 전용 — 위 조건이 충족된 경우에만 적용]'
)
WHERE key IN ('story_rules','multiStory_rules');
SQL
```

- [ ] **Step 3: 검증** — 게이팅 줄 제거 + 신규 블록 + [성애 장면 전용] 1회 유지 확인:

```bash
docker exec storyfit-db-1 psql -U storyfit -d storyfit -tA -c "SELECT key,
  (value LIKE '%안전한 상황에서 명확한 욕망%') AS gate_left,
  (value LIKE '%로맨스 단계에서는 노골%') AS romance_gate_left,
  (value LIKE '%관계 맥락 — 톤을 정하는 참고%') AS has_context,
  (value LIKE '%친밀 장면 속도·전희%') AS has_foreplay,
  (length(value) - length(replace(value, '[성애 장면 전용', ''))) / length('[성애 장면 전용') AS sexonly_count
FROM \"GlobalConfig\" WHERE key IN ('story_rules','multiStory_rules') ORDER BY key;"
```
Expected: `gate_left=f`, `romance_gate_left=f`, `has_context=t`, `has_foreplay=t`, `sexonly_count=1`.

> 만약 `gate_left=t`(제거 실패)면 backup으로 복구(`UPDATE ... SET value = '<백업값>'`)하고 정규식을 실제 줄바꿈에 맞춰 재시도.

---

### Task 4: 가이드 동기화 + 최종 검증

**Files:**
- Modify: `app/(main)/guide/page.tsx`

- [ ] **Step 1: 가이드 항목 추가** — `⚙️ 대화 설정` 섹션(안전 수준 항목 근처)에:

```ts
      { emoji: '🔞', label: '성인 합의 게이팅', desc: 'ON이면 성애 장면 진입에 합의·맥락 전제를 둡니다. OFF면 진입 게이팅을 풀어 더 자유롭게 전개합니다(미성년 등 모델 자체 안전선은 항상 유지). 채팅 사이드패널 AI응답 탭에서 켜고 끕니다.' },
```

- [ ] **Step 2: 전체 검증** — Run: `npx tsc --noEmit && npx vitest run` → exit 0, 전부 PASS.

- [ ] **Step 3: 커밋**

```bash
git add "app/(main)/guide/page.tsx"
git commit -m "docs: 성인 합의 게이팅 기능 가이드 동기화"
```

---

## 배포 (누적 변경과 함께, 사용자 확인 하에)

```bash
cd apps/web && git push origin HEAD:main
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (성애 프롬프트 개선)" && git push origin master
```
DB(Task 3)는 라이브 반영됨. 코드 배포 후 `docker compose up --build -d`.

## Self-Review 메모

- 스펙 커버리지: 게이팅 토글(T1·T2 + DB 줄 제거 T3), 관계맥락·전희(T3 DB), 가이드(T4) — 전부 매핑.
- 타입 일관성: `adultGating`/`adultGatingEnabled` 전 태스크 일관(fastPace와 동형).
- ⚠️ T3 정규식 `E'\n  - 성애 장면은 서사가.*?시작한다'`는 story/multi 줄바꿈 차이를 's' 플래그(. 가 개행 포함)+비탐욕(?)으로 흡수. 검증 SELECT로 반드시 확인.
- ⚠️ T2 토글 마크업은 fastPace 토글과 동일 클래스로 맞출 것.
