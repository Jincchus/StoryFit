# 프롬프트 3종 개선 설계 (빠른전개 토글 · 길이 min/max · 멀티 선택지)

작성일: 2026-06-27
대상(코드): `prisma/schema.prisma`, `lib/systemPrompt.ts`, `lib/chatPipeline.ts`, `app/api/conversations/[id]/{chat,regenerate,continue}/route.ts`, `app/api/conversations/[id]/route.ts`, `types/index.ts`, `app/(main)/conversations/new/_components/StyleSection.tsx`, `app/(main)/conversations/[id]/_components/SidePanel.tsx`, `app/(main)/guide/page.tsx`
대상(DB·GlobalConfig): `story_rules`, `multiStory_rules`, `multiStory_closing`

> **핵심 구조**: `modeRules`(story_rules/multiStory_rules)·`closing`(multiStory_closing)은 코드가 아니라 **DB(GlobalConfig)** 에 저장된다. 따라서 규칙 텍스트 수정은 git 배포가 아니라 **라이브 DB에 SQL UPDATE**로 적용한다(사용자 승인 완료). 코드는 git, 규칙은 DB — 두 경로 분리.

## 배경

직전 가상 테스트에서 확인한 사항:
- 스토리/멀티 base는 "장면을 전개하라"고 지시하나, modeRules는 "느리게·결론 서두르지 않음"으로 감속 + "본문 600~800자" 고정.
- `styleConfig`에 `length`(짧게/보통/길게)·`pace`(빠름/보통/느림) 이미 존재. `length`는 enum.
- 멀티는 base·closing 둘 다 "선택지 금지" → 선택지 미출력. 프런트(`MessageList`)는 이미 `isStoryOrMulti`로 스토리·멀티 선택지 파싱·렌더 지원(멀티는 프롬프트만 막혀 있었음).

## 목표 (사용자 확정)

1. **빠른 전개 토글** — 채팅창 전용 신규 on/off(기본 off) + 강한 빠른전개 프롬프트(감속 지침 오버라이드).
2. **응답 길이 min/max** — `styleConfig.length`를 자 수 `{min?,max?}`로 교체(기본 비움), modeRules의 "600~800자" 제거, 기존 enum값은 버림.
3. **멀티 선택지** — 멀티도 `--- + 4지선다` 출력. 1·2번=페르소나 행동/대사, 3·4번=캐릭터 대사/행동(이번 응답 등장 캐릭터 기본, 흐름상 새 캐릭터 등장 필요 시 가능). "미등장 캐릭터 제외" 제약은 두지 않음.

## 설계

### #1 빠른 전개 토글

- **스키마**: `Conversation.fastPaceEnabled Boolean @default(false)` (prisma db push).
- **프롬프트**: `buildStorySystemPrompt`/`buildMultiStorySystemPrompt`에 `fastPace?: boolean` 추가. true면 **modeRules 블록 직후**(정적 프리픽스 내)에 아래 블록을 push:

```
[전개 속도 — 다른 모든 속도 지시보다 우선]
- 이번 대화는 빠르게 전개한다: 시간·장소를 과감히 건너뛰고, 한 응답에서 사건을 여러 단계 진행시켜 상황을 크게 움직인다.
- 군더더기 묘사·뜸들이는 서술을 줄이고 핵심 전개에 집중한다.
- 이 지시는 [공통 문체]의 "느리게 서술"·"결론을 서두르지 않는다"보다 우선한다.
```

- **연결**: `buildModeSystemPrompt`(chatPipeline)에 `fastPace` 인자 추가 → 빌더 전달. chat/regenerate/continue 3개 라우트에서 `fastPace: conv.fastPaceEnabled ?? false` 전달(기존 `flipPersonaPlaceholders`·`allowPersonaDialogue`와 동일 패턴).
- **UI**: 채팅 `SidePanel`에 토글 추가(기존 `personaAutoMode` 토글 패턴 복제, 기본 off). `app/api/conversations/[id]/route.ts` PATCH 허용목록에 `fastPaceEnabled` 추가.
- 기존 `styleConfig.pace=빠름`은 유지(누적 가능, 충돌 아님).
- 시스템 프롬프트 정적/가변 분리 불변: 이 블록은 per-conversation 정적값이라 프리픽스(modeRules 직후)에 둬 캐싱 유지.

### #2 응답 길이 min/max

- **타입**(`types/index.ts`): `length?: '짧게'|'보통'|'길게'|null` → `length?: { min?: number; max?: number } | null`.
- **buildStyleSection**(`lib/systemPrompt.ts`): length가 객체일 때만 출력. 레거시 문자열은 무시(타입 가드).

```ts
if (s.length && typeof s.length === 'object') {
  const { min, max } = s.length
  if (min && max) lines.push(`- 응답 길이: ${min}~${max}자`)
  else if (min)   lines.push(`- 응답 길이: 최소 ${min}자 이상`)
  else if (max)   lines.push(`- 응답 길이: 최대 ${max}자 이내`)
}
```

- **modeRules(DB)**: `story_rules`·`multiStory_rules`에서 "  - 본문은 600~800자로 서술한다" 줄 제거(SQL).
- **UI**(`StyleSection.tsx`): 길이 칩 행을 최소/최대 숫자 입력 2개로 교체(빈 값 허용 → 해당 항목 미적용). 다른 곳에서 length 칩을 렌더하면 동일 처리(구현 시 grep 점검).
- 기존 대화의 문자열 length값은 buildStyleSection 타입가드로 무시되어 길이 지시가 안 들어갈 뿐, 오류 없음.

### #3 멀티 선택지

- **멀티 base(하드코딩, `lib/systemPrompt.ts`)**: `- Do NOT offer choices, numbered options, or host-like questions. Never append a "---" divider followed by a list of options. End with the scene itself.` 줄 제거(스토리에서 한 것과 동일).
- **multiStory_closing(DB)**: "Never offer choices…" 제거하고 선택지 규칙 추가. 본문의 페르소나 미서술 규칙은 유지. 새 값:

```
[Top Priority]
- Always respond in Korean only
- Never output analysis, planning, or meta-commentary
- Begin directly with scene, dialogue, or action
- In the body, never write the persona(user)'s words, actions, emotions, or decisions
- Characters perform only their own words and actions, then naturally advance the scene
- Write richly with narration, action, and dialogue
- REQUIRED: Do not repeat dialogue, actions, or narration already present in the previous exchange
- At the end, place a "---" divider and present 4 numbered choices:
  - Choices 1–2: the persona(user)'s next action or dialogue
  - Choices 3–4: a character's next dialogue or action (prefer characters present in this response; a new character may appear if the flow naturally calls for one)
```

- **프런트**: 변경 불필요(이미 `isStoryOrMulti`로 파싱·렌더).
- 스토리(`story_closing`)는 변경하지 않음.

## 영향 / 리스크

- **DB SQL은 라이브 반영**: 적용 즉시 전체 사용자 대화에 영향. 적용 전 현재 값 백업(SELECT 저장), 적용 후 SELECT로 검증.
- prisma db push로 컬럼 추가(backward-compatible, default false).
- 시스템 프롬프트 조립 순서 불변(CLAUDE.md): 빠른전개 블록만 modeRules 직후(정적)로 추가, 나머지 순서 불변.
- guide 동기화: 빠른전개 토글·길이 설정은 사용자 체감 기능 → `guide/page.tsx` 갱신.

## 검증

- `npx tsc --noEmit` exit 0, `npx vitest run`(기존 + buildStyleSection 길이 단위 테스트 추가 가능).
- DB: 적용 후 `SELECT`로 600~800자 줄 제거·multiStory_closing 선택지 규칙 반영 확인.
- 수동: 멀티 대화에서 선택지 4개(1·2 페르소나 / 3·4 캐릭터) 출력, 빠른전개 토글 on 시 전개 가속 체감.

## 범위 외 (YAGNI)

- 스토리 선택지 규칙 변경(이미 동작), styleConfig의 pace/length 외 항목 변경, 선택지 개수 가변화.
