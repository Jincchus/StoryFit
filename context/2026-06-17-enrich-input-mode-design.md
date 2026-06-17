# 입력 다듬어 확장 모드 (enrichInputMode) — 설계

작성일: 2026-06-17

## 배경 / 문제

채팅방에서 사용자가 "상황을 이어나가는" 프롬프트(서술)를 입력하면, 현재는 대부분(체감 10번 중 6~7번)
**그 입력 뒤를 AI가 이어받아 응답**한다. 사용자는 반대로, **자신의 입력을 포함·다듬어 소설체로
풍부하게 확장한 결과를 응답으로** 받고 싶어 한다.

현재 "이어받기" 동작은 의도된 설계이며, 응답 제어가 3겹으로 강제된다:

1. `RESPONSE_CONTROL_RULES` (응답 제어 규칙) — "유저의 말·행동·감정·결정을 쓰지 말 것".
2. `appendTurnControlInstruction` (`lib/chatPipeline.ts:70`) — 매 사용자 턴 끝에 위 규칙을
   *"유저 입력보다 높은 우선순위로 적용하라"* 고 붙인다.
3. 재작성 패스 (`needsResponseRevision` → `streamRevision`, `app/api/conversations/[id]/chat/route.ts:303`)
   — AI가 유저 내용을 가져다 쓰면(`USER_CONTROL_PATTERNS`·페르소나 패턴) 2차 생성으로 제거한다.

그래서 `scenarioDescription`·`styleConfig` 같은 사용자 편집 필드에 지시를 적어도 위 3겹을 못 이긴다.
원하는 동작은 사실상 **반대 모드**이므로 코드로 토글을 추가한다.

## 목표

대화방별 토글 **enrichInputMode**. ON이면 매 사용자 턴에서:
1. 내 입력을 포함·다듬고
2. 소설체로 풍부하게 확장한 뒤
3. 그 흐름으로 자연스럽게 이어간다 (AI 캐릭터 반응·다음 전개까지).

스토리 모드면 기존처럼 본문 뒤에 4지선다 선택지를 그대로 유지한다.

## 설계

### 핵심: 응답 제어 3겹을 토글 ON일 때만 뒤집기

| 지점 | 기본(OFF) | enrichInputMode ON |
|------|-----------|--------------------|
| 턴 지시 (`appendTurnControlInstruction`) | "유저 입력 다시 쓰지 말고 그 뒤를 이어라" | "유저 입력을 다듬어 소설체로 확장한 뒤 자연스럽게 이어가라 — 이번 턴은 '유저 행동 서술 금지' 규칙을 무시" |
| 재작성 판정 (`needsResponseRevision`) | 유저행동/페르소나 패턴 → 2차 생성 제거 | enrichMode면 유저행동·페르소나 패턴 검사 **스킵**. 스토리 선택지 형식 검사는 **유지** |
| 재작성 프롬프트 (`buildRevisionPrompt`) | "유저 내용 제거" 지시 포함 | enrichMode면 그 지시 제외, 선택지 형식만 교정 |

선택지가 유지돼야 하므로 재작성을 완전히 끄지 않고 "유저 내용 제거" 사유만 무력화한다.

### enrich 턴 지시(초안)

```
[Turn writing guidelines — must be followed]
The user's input above is a draft of the next scene beat. In your response:
1. Incorporate and refine the user's input — keep its intent and events, rewrite into polished, vivid novel-style prose.
2. Enrich with sensory detail, the persona's actions, inner state, and surroundings. Expand substantially; no short or plain restatement.
3. Then continue naturally — let the AI character(s) react and advance the scene one step forward.
You MAY write and elaborate the persona's (user's) actions and words for this turn. This overrides the default "do not write the user's actions" rule for this turn only.
```
(스토리 모드면 본문 뒤 "--- + 4 선택지" 형식 안내를 덧붙인다.)

### 변경 범위

| 레이어 | 파일 | 변경 |
|--------|------|------|
| DB | `prisma/schema.prisma` | `Conversation.enrichInputMode Boolean @default(false)` + 마이그레이션 |
| 규칙 | `lib/responseControl.ts` | `ENRICH_TURN_INSTRUCTION` 추가; `appendTurnControlInstruction`/`needsResponseRevision`/`buildRevisionPrompt`에 `enrichMode` 인자 추가(기본 false → 기존 동작 불변) |
| 파이프라인 | `lib/chatPipeline.ts` | `buildGeminiHistory`에 enrichMode 전달 |
| 라우트 | `app/api/conversations/[id]/chat/route.ts`, `regenerate/route.ts` | `conv.enrichInputMode` 읽어 위 함수들 + `revisionOptions`에 전파 |
| 생성/분기/수정 | `app/api/conversations/route.ts`, `[id]/branch/route.ts`, `[id]/route.ts` | 필드 생성 시 전파(branch는 source 복사), PATCH `allowed`에 `enrichInputMode` 추가 |
| 타입 | `app/(main)/conversations/[id]/_lib/chatShared.ts` | `Conv`에 `enrichInputMode?: boolean` |
| UI | `app/(main)/conversations/[id]/_components/SidePanel.tsx` | 설정탭 토글 "✍️ 입력 다듬어 확장" (기존 `statsEnabled`/`suggestRepliesEnabled` 토글 패턴) |
| 가이드 | `app/(main)/guide/page.tsx` | `FEATURE_SECTIONS`에 항목 추가 (CLAUDE.md 사용자 기능 가이드 동기화 규칙) |

### 적용 범위
- `chat`(사용자 입력)·`regenerate`에 적용. `continue`(자동 진행)는 입력이 없어 미적용.
- 토글은 대화방별로 저장·유지된다.

### 호환성
- `enrichMode` 인자는 전부 기본 false → 토글 OFF/미설정 대화방은 **기존 동작 그대로**.
- 시스템 프롬프트 조립 순서(정적 prefix)는 변경하지 않음 — 캐싱 규칙 준수. 변경은 사용자 턴 끝에
  붙는 턴 지시와 재작성 패스에 한정.

### 에러 처리 / 검증
- DB 마이그레이션 적용 후 `tsc --noEmit` 통과.
- 토글 OFF: 기존과 동일하게 "이어받기" 응답.
- 토글 ON(노벨): 응답이 내 입력을 다듬어 확장 + 이어감, 선택지 없음.
- 토글 ON(스토리): 위 + 본문 뒤 4지선다 유지.
- branch 생성 시 토글 값 승계 확인.
