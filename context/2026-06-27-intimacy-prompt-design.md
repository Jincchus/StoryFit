# 성애/친밀 장면 프롬프트 개선 설계 (관계 맥락 · 전희 디테일 · 합의 게이팅 토글)

작성일: 2026-06-27
대상(코드): `prisma/schema.prisma`, `lib/systemPrompt.ts`, `lib/chatPipeline.ts`, `app/api/conversations/[id]/{chat,regenerate,continue}/route.ts`, `app/api/conversations/[id]/route.ts`, `app/api/conversations/route.ts`, `app/(main)/conversations/[id]/_components/SidePanel.tsx`, `app/(main)/guide/page.tsx`
대상(DB·GlobalConfig): `story_rules`, `multiStory_rules`

> 성인 롤플레이 픽션의 기존 기능 개선. **성인 캐릭터·합의 기반** 범위. 미성년·실제 비합의·CSAM 같은 하드리밋은 우리 프롬프트에 없고 모델(Gemini) 자체 안전선이 강제한다 — 본 작업은 그 하드리밋을 건드리지 않으며, 어떤 토글도 모델 안전 우회 문구를 추가하지 않는다.

## 배경 / 문제

- 현재 성애 규칙은 `modeRules`(DB)의 `[장면 분위기 판단]` + `[성애 장면 전용]`에 있음.
- 문제 1: **절정까지 너무 빠름** — 전희·애무에 분량을 쓰라는 지시가 없고 절정 지향 문구만 있음.
- 문제 2: **이중 게이팅** — 우리 프롬프트의 소프트 게이팅("안전한 상황에서 명확한 욕망 교환 시에만 시작", "로맨스 단계에선 노골 금지")이 모델 안전선과 겹쳐 사소한 것도 막음.
- 문제 3: **선형 단계 가정** — 친밀/연인이어야 성애 가능하다는 식이면 원나잇·"호감 단계 잠자리 후 연애" 등이 막힘.

## 목표 (사용자 확정)

1. **관계 맥락** = 진입 게이트가 아니라 톤(감정 결)을 정하는 참고. 어떤 관계에서도 성애 가능, 맥락에 따라 묘사 결만 다름.
2. **전희·속도·부위별 디테일 강화** = 절정으로 직행하지 않고 부위별로 차례로 공들여 묘사.
3. **합의 게이팅 on/off 토글** = 우리 소프트 게이팅을 끌 수 있게. 기본 ON(현 동작 유지). OFF는 우리 게이팅을 "안 넣을 뿐"(우회 문구 없음).

## 설계

성애 규칙을 두 갈래로 분리:
- **always-on(품질·톤)** → `modeRules`(DB) 편집
- **toggle-gated(진입 전제)** → 코드 상수, `adultGatingEnabled`일 때만 주입(fastPace 패턴)

### A. always-on 블록 (DB `story_rules`·`multiStory_rules` 편집)

기존 `[장면 분위기 판단]`에서 **아래 두 줄 제거**:
- "로맨스의 긴장과 성애는 다르다. 로맨스 단계에서는 노골적 신체 묘사를 사용하지 않는다" → 전면 제거(단계는 게이트 아님)
- "성애 장면은 … 안전한 상황에서 명확한 욕망을 주고받았을 때, 또는 사용자가 그 방향으로 이끌 때만 시작한다" → 코드 게이팅 블록으로 이동(B)

그 외 비게이팅 줄(감정 판단, 공포≠성적 해석 등)은 유지. 아래 두 블록을 `[성애 장면 전용]` 앞에 추가:

```
[관계 맥락 — 톤을 정하는 참고(진입을 막는 조건이 아님)]
- 두 인물의 현재 관계 맥락을 판단해(답변엔 미포함) 친밀 장면의 심리·태도·온도를 거기에 맞춘다.
  예: 낯선 사이·원나잇 → 낯섦·긴장·즉흥적 욕망 / 호감 → 설렘·탐색·서툶 / 연인·부부 → 애정·익숙함·신뢰
- 관계 단계는 성애 진입을 막는 게이트가 아니다. 원나잇, 호감 단계의 잠자리, 이후 연애로의 발전 등 어떤 전개도 가능하다.
- 같은 행위라도 관계 맥락에 따라 감정의 결과 대사 톤을 다르게 표현한다.

[친밀 장면 속도·전희 — 절정보다 우선]
- 절정으로 곧장 가지 않는다. 분위기 조성 → 접촉·애무(전희) → 점진적 고조 순으로 천천히 진행한다.
- 전희를 한 덩어리로 뭉뚱그리지 않는다. 손길·입술이 닿는 부위를 하나씩 따라가며(목덜미·쇄골·가슴·허리·허벅지 등) 각 부위의 애무를 차례로 구체적으로 묘사한다.
- 한 부위에 충분히 머문 뒤 다음 부위로 옮겨가고, 매 동작마다 감각(촉감·온도·압력·소리)과 상대의 반응(떨림·호흡·목소리)을 짝지어 서술한다.
- 응답 분량의 대부분을 전희·애무에 쓴다. 절정은 장면의 전희가 충분히 무르익고 사용자가 그 방향으로 명확히 이끌 때 다다른다.
- 다음 수위로 넘어갈 때마다 상대의 반응을 살피며 강도를 한 단계씩 올린다.
- 친밀 장면에서는 이 전희·속도 규칙이 [전개 속도] 빠른 전개 지시보다 우선한다.
```

기존 `[성애 장면 전용]` 디테일 규칙은 유지.

### B. toggle-gated 블록 (코드 `lib/systemPrompt.ts`)

상수 `INTIMACY_GATING_BLOCK`:

```
[성애 진입 — 합의·맥락 전제]
- 성애 장면은 서사가 자연스럽게 그 방향으로 흐르고 두 인물이 명확한 욕망을 주고받았을 때, 또는 사용자가 그 방향으로 명확히 이끌 때 시작한다.
- 한쪽이 거부·주저하는 신호를 보이면 그 의사를 존중해 진행을 멈추거나 속도를 늦춘다.
```

- `adultGating === true`(기본)일 때만 modeRules 직후 주입(fastPace와 동일 위치).
- `false`면 주입하지 않음 — 그 외 어떤 우회/무시 문구도 넣지 않는다. 모델 자체 안전선·하드리밋은 토글과 무관하게 유지.

### C. 스키마·배선·UI (fastPace 패턴 복제)

- `Conversation.adultGatingEnabled Boolean @default(true)` (ALTER, 컬럼 추가)
- `buildStorySystemPrompt`/`buildMultiStorySystemPrompt`에 `adultGating?: boolean`(기본 true) → true면 `INTIMACY_GATING_BLOCK` 주입
- `buildModeSystemPrompt`(chatPipeline)에 `adultGating` 인자 + chat/regenerate/continue 3개 라우트에서 `adultGating: conv.adultGatingEnabled ?? true` 전달
- `app/api/conversations/[id]/route.ts` PATCH 허용목록에 `adultGatingEnabled`, POST 수용
- `SidePanel`에 토글 `🔞 성인 합의 게이팅`(기본 ON, OFF면 우리 소프트 게이팅 해제 안내 문구)

## 검증

- `npx vitest run`(systemPrompt에 adultGating ON/OFF 블록 유무 단위 테스트 추가) + `npx tsc --noEmit`.
- DB: 적용 후 SELECT로 (a) 두 게이팅 줄 제거 (b) 관계맥락·전희 블록 추가 확인.
- 수동: 게이팅 ON/OFF별 진입 차이, 전희가 부위별로 길게 나오는지.

## 영향 / 리스크

- DB SQL 라이브 반영(백업 후). modeRules 편집은 전 사용자 영향.
- ALTER 컬럼 추가(backward-compatible, default true).
- 시스템 프롬프트 조립 순서 불변: 게이팅 블록만 modeRules 직후(fastPace와 같은 정적 위치).

## 범위 외 (YAGNI)

- 관계 단계의 스탯/수치 연동, 사용자 수동 단계 설정 UI(이번엔 AI 자동 판단).
- 하드리밋 관련 어떤 변경도 없음.
