# 게임모드(Game Mode) 설계 — 롤플레이/소설 모드 제거 + 게임모드 추가

## 배경 및 목표

기존 대화 모드 중 `roleplay`와 `novel`을 프론트엔드/백엔드 양쪽에서 완전히 제거하고,
그 자리에 새로운 `game` 모드를 추가한다.

게임모드는 BJ 방송 챗봇류 서비스에서 흔히 보이는 `!상태창`/`!캐릭터이름`/`!엔딩` 같은
고정 슬래시 커맨드 + 시스템 카드 + 관계 점수 시스템을 묶은 모드다. 관계 점수는 매
응답마다 AI가 스스로 판단해 증감시키고, 누적된 점수에 따라 유저가 원할 때
`!엔딩`으로 캐릭터별 개별 엔딩을 받아볼 수 있다.

## 범위 (Scope)

- 제거: `roleplay`/`novel` 모드 관련 코드 전체 (시스템 프롬프트 빌더, 라우트 분기,
  프론트엔드 렌더링/선택 UI), 및 DB의 해당 모드 대화방 3건 삭제
- 추가: `game` 모드 (멀티스토리 렌더링 패턴 기반 + 슬래시 커맨드 + 카드 UI + 관계
  점수 시스템 + 생성 시점 커스터마이징)

## 기존 데이터 확인

조회 결과, DB에 `roleplay` 2건, `novel` 1건의 대화방이 존재한다 (2026-05-20~21,
유저 2명):

| mode | title | userId | updatedAt |
|---|---|---|---|
| roleplay | 세드릭 윈터가드와의 대화 | cmpfipekk... | 2026-05-21 |
| novel | 테온와의 대화 | cmpe3odui... | 2026-05-20 |
| roleplay | c와의 대화 | cmpe3odui... | 2026-05-20 |

→ **결정: 3건 모두 삭제한다** (사용자 확인 완료).

---

## 1. 제거 범위 (Roleplay/Novel 제거)

### 백엔드
- `lib/systemPrompt.ts`: `buildSystemPrompt`(roleplay), `buildNovelSystemPrompt`(novel)
  와 그에 종속된 미사용 파라미터 인터페이스/헬퍼 삭제
- `app/api/conversations/[id]/chat/route.ts`,
  `app/api/conversations/[id]/regenerate/route.ts`: `conv.mode === 'novel'` /
  roleplay 분기 제거 — `systemPrompt` 선택, `personalRules` 선택,
  `triggerStateTracking` 호출부 ("롤플레이/소설 모드 씬 상태 자동 추적" — 현재
  roleplay/novel에서만 트리거되는 유일한 후처리)
- `app/api/conversations/route.ts`: 기본 모드 폴백을 `'roleplay'` → `'story'`로
  변경 (현재 `mode: body.mode ?? 'roleplay'`)
- `prisma/schema.prisma`: `Conversation.mode String @default("roleplay")` →
  `@default("story")` 로 변경, 마이그레이션 실행
- `app/api/conversations/generate-scenario/route.ts`: `'롤플레이 (1:1 대화)'`
  modeLabel 항목 제거

### 프론트엔드
- `app/(main)/conversations/new/page.tsx`: 이미 주석 처리된 roleplay/novel 버튼
  (라인 408-409) 삭제, `mode` 유니온 타입에서 `'roleplay' | 'novel'` 제거,
  관련 라벨 텍스트 정리
- `app/(main)/conversations/[id]/page.tsx`: `isNovel`, `isTikiTaka` 산출 시
  roleplay 폴백 제거, 이를 참조하는 모든 렌더링 분기 (버블 스타일, placeholder,
  모드 배지 등 — 약 라인 581-1025) 정리
- `app/(main)/library/[id]/page.tsx`: `useProseLayout = conv.mode === 'novel' ||
  isStory` → `isStory`로 단순화
- `app/(main)/chatlist/page.tsx`: 이미 주석 처리된 roleplay/novel 필터 옵션의
  죽은 주석 정리
- `app/(main)/admin/prompts/page.tsx`: `'roleplay' | 'novel'` 탭 제거 (내부
  프롬프트 점검용 도구이므로 단순 삭제)

### 데이터베이스
- 기존 roleplay 2건 / novel 1건 대화방 삭제
- 스키마 변경에 대한 Prisma 마이그레이션 실행

---

## 2. 게임모드 데이터 모델

새 `mode = 'game'` 값을 추가한다. 기존 `statsConfig`가 정적 정의(name/min/max)와
가변 현재값(value)을 하나의 JSON 배열에 함께 저장하는 패턴을 그대로 따른다 — 별도의
config/state 테이블 분리 없이 단일 소스를 유지한다.

### `Conversation` 모델 — 추가 필드
```prisma
gameConfig  Json?   // GameScoreEntry[] — 생성 시점에 설정, 평가에 의해 갱신됨
```

`GameScoreEntry` 형태:
```ts
type GameScoreEntry = {
  characterId: string
  scoreName: string                 // 예: "호감도" — 생성 시 유저 커스터마이징
  value: number                     // 현재값 (매 AI 응답 후 갱신됨)
  min: number
  max: number
  criteria: string                  // 자유 텍스트: "무엇이 점수를 올리고 내리는가"
  endings: {
    threshold: number
    op: '>=' | '<='
    label: string                   // 짧은 엔딩명, 예: "해피엔딩"
    description: string             // 해당 엔딩 생성 시 AI에게 줄 가이드
  }[]
}
```

### `Message` 모델 — 추가 필드
```prisma
gameDelta  Json?   // Record<characterId, number> — 회귀생성/삭제 시 롤백용 (statsDelta 패턴)
cardType   String? // 'status' | 'character' | 'ending' | null
cardData   Json?   // 카드 UI용 구조화 페이로드; content는 일반 텍스트 폴백 요약
```

`gameEnabled` 같은 별도 토글은 두지 않는다 — `mode === 'game'` 자체가 게임
파이프라인 적용 여부를 의미하므로 (stats/inventory처럼 story/multiStory 내부의
선택적 부가기능이 아니라 모드 자체이기 때문).

`content`를 그대로 사람이 읽을 수 있는 텍스트로 유지하고 `cardType`/`cardData`를
구조화 필드로 분리함으로써, 렌더러는 `if (message.cardType)` 만 체크하면 되고
요약/검색/내보내기에서도 자연스럽게 동작한다.

---

## 3. 채팅 파이프라인 — 시스템 프롬프트, 슬래시 커맨드, 렌더링

### 시스템 프롬프트 (`buildGameSystemPrompt`)
`systemPrompt.ts`에 `buildMultiStorySystemPrompt`의 가까운 형제로 추가한다. 동일한
`[Output Format]` 규칙(Name : "대사", Name : '속마음', 멀티 캐릭터 보이스 규칙)을
따르되 선택지(choices) 섹션은 없음 — 게임모드는 선택지가 아닌 커맨드로 진행되므로.
대신 다음 섹션을 추가한다:

```
[관계 점수 — 내부 판단용, 화면에 직접 언급 금지]
{character.name}: {scoreName} 현재 {value}/{max}
판단 기준: {criteria}
```

항상 `characters: Character[]` 배열을 받는다 (길이 1 이상) — 단일/멀티 캐릭터
대화를 하나의 코드 경로로 처리하여 "멀티스토리 렌더링을 따른다"는 요구를 일관되게
충족한다. CLAUDE.md의 고정된 시스템 프롬프트 조립 순서에서 (4) 단계
(Character systemPrompt+scenarioDescription)에 `buildMultiStorySystemPrompt`와
동일한 위치에 들어간다 — 순서 변경 없음.

### 슬래시 커맨드 감지 (`chat/route.ts`, 일반 생성 호출 이전)
```
trimmed = userMessage.trim()
if trimmed.startsWith('!'):
    cmd = trimmed.slice(1).trim()
    if cmd === '상태창'      → handleStatusCard()
    elif cmd === '엔딩'      → handleEndingCard()
    elif cmd이 conv.characters 중 한 캐릭터 이름과 일치 → handleCharacterCard(matchedCharacter)
    else → 일반 채팅으로 폴백 (대사 중 "!" 가 들어가도 깨지지 않도록)
```

각 핸들러는:
1. 전용 프롬프트로 AI에게 구조화 JSON을 요청 (storyEval.ts/import 라우트에서 이미
   쓰는 `extractJson` / `raw.match(/\{[\s\S]*\}/)` 패턴 재사용)
2. 결과를 `role: 'assistant'`, `cardType`, `cardData`, 그리고 일반 텍스트 요약
   `content`를 가진 `Message`로 저장
3. 스트리밍 없이 즉시 반환 — **점수 평가(triggerGameEvaluation)는 트리거하지 않음**
   (커맨드는 내러티브 턴이 아닌 메타 액션이므로)

### 카드별 데이터 형태
- `상태창` → `{ relationships: [{characterId, name, scoreName, value, max, situationSummary}] }`
  — 캐릭터별 한 행
- `캐릭터이름` → `{ characterId, name, summary, currentDisposition }` — 현재 점수와
  최근 맥락에 기반해 그 캐릭터가 유저를 현재 어떻게 느끼는지에 대한 짧은 AI 생성 요약
- `엔딩` → `{ endings: [{characterId, name, label, narrative}] }` — **모든** 캐릭터에
  대해, 각자의 현재 `value`와 `endings` 임계값 목록 중 가장 부합하는 것을 AI가
  선택해 캐릭터별 짧은 엔딩 내러티브를 생성. 어떤 임계값도 충족하지 못하면
  "관계가 아직 무르익는 중" 류의 기본 내러티브로 대체

### 렌더링 (프론트엔드)
`conversations/[id]/page.tsx`의 메시지 리스트에서 (기존 멀티스토리 버블 로직과
나란히) `message.cardType`을 먼저 체크 — 있으면 일반 버블/`parseNovelBlocks` 경로
대신 전용 `<GameCard type={cardType} data={cardData} />` 컴포넌트를 렌더링한다.
`cardType`이 없는 일반 메시지(유저/어시스턴트)는 오늘의 멀티스토리 버블과 동일하게
렌더링된다.

---

## 4. 점수 평가, 롤백, 생성 시점 커스터마이징

### 점수 평가 (`triggerGameEvaluation`, `storyEval.ts`에 `triggerStoryEvaluation`의
형제로 추가)
일반(슬래시 커맨드가 아닌) AI 응답마다 트리거된다:
1. 모든 캐릭터의 `scoreName`/`criteria`/현재 `value`를 나열한 통합 프롬프트로
   `{ "deltas": { [characterId]: number, ... } }` 형태의 JSON을 요청 (델타 범위
   -5~+5, 기존 `extractJson` 파싱 패턴 재사용)
2. 캐릭터별로 `newValue = clamp(value + delta, min, max)` 계산 후
   `Conversation.gameConfig` 배열에 제자리 갱신
3. `{ [characterId]: delta }`를 `Message.gameDelta`에 저장 (롤백용)
4. `triggerStoryEvaluation`과 동일하게 fire-and-forget 비동기 실행 — 응답
   스트림을 막지 않음

### 롤백 (`rollbackGameDelta`, `rollbackStatsDelta`의 형제)
각 캐릭터의 델타를 역적용하고 재클램핑. 기존에 `rollbackStatsDelta`/
`rollbackInventoryDelta`를 호출하는 동일한 두 지점에 연결:
- `regenerate/route.ts` (마지막 어시스턴트 메시지를 폐기하고 재생성하기 전)
- `messages/route.ts` (메시지 삭제 시)

### 생성 시점 커스터마이징 UI (`conversations/new/page.tsx`)
`mode === 'game'`일 때, 선택된 캐릭터별로 패널 표시 (기존 stats 태그 선택 패널과
유사한 UI 패턴):
- 점수 이름 (텍스트 입력, 기본값 "호감도")
- 범위 min/max (숫자 입력, 기본값 0~100, 초기값 = 중간값)
- 판단 기준 (텍스트영역 — 점수를 올리고 내리는 요인을 자유 서술, `criteria`로 전달)
- 엔딩 임계값 목록 (작은 반복 가능한 리스트: 임계값, 라벨, 설명) — 합리적인 프리셋
  제공 (예: "80 이상 → 해피엔딩", "20 이하 → 새드엔딩") 후 유저가 수정/추가 가능.
  `op`("이상"/"이하")은 별도 입력란 없이 프리셋의 표현("이상"→`>=`, "이하"→`<=`)에서
  자동으로 결정되어 UI를 단순하게 유지한다

이것이 커스터마이징의 전부다 — 커스텀 커맨드 작성 기능은 없음 (유저가 "가벼운"
수준으로 확정).

### `!엔딩` 실행 후 대화 지속 여부
엔딩 카드는 단순히 "현재 점수 기준 스냅샷 내러티브"이며, 대화를 종료/잠그지 않는다.
유저는 엔딩을 본 뒤에도 계속 대화를 이어갈 수 있고, 점수가 더 변하면 다시
`!엔딩`을 호출해 달라진 결과를 확인할 수 있다 — "마지막에 유저가 엔딩을 내는
걸로 하자"는 요구를 "유저가 원할 때 몇 번이든 확인 가능"으로 해석한 것이다.

---

## 5. 에러 처리 및 엣지 케이스

- **AI가 잘못된 JSON 반환** (카드 생성 / 점수 평가): `evalStory`의 2회 재시도
  루프를 그대로 따름. 카드 핸들러는 실패 시 `cardType: null`의 안내 텍스트
  카드("지금은 카드를 불러올 수 없어요")로 폴백; 점수 평가는 오늘의 `evalStory`가
  `null`을 반환할 때처럼 조용히 갱신을 건너뜀 (DB 쓰기 없음)
- **`!캐릭터이름`이 어떤 캐릭터와도 일치하지 않음**: 일반 채팅으로 폴백 (3절에서
  이미 다룸) — "상태창"이라는 이름의 캐릭터가 있는 경우 같은 false negative 방지
- **`!엔딩` 호출 시 어떤 임계값도 충족하지 못한 캐릭터**: AI가 "관계가 무르익는 중"
  류의 기본 내러티브를 작성 — 게임이 막다른 길로 가지 않고 계속 진행 가능
- **빈 `gameConfig`** (이 기능 추가 이전 생성된 대화, 또는 커스터마이징 생략):
  `buildGameSystemPrompt`는 `[관계 점수]` 섹션을 생략하고
  `triggerGameEvaluation`/슬래시 핸들러는 아무 동작도 하지 않음 — 게임모드는
  순수 멀티스토리 스타일 내러티브 채팅으로 동작
- **스트림 중간 중단**: 기존 partial-save 동작(CLAUDE.md 규칙)은 그대로 유지 —
  `triggerGameEvaluation`은 오늘의 `triggerStoryEvaluation`처럼 응답이 정상
  완료된 경우에만 트리거됨

## 6. 테스트 계획

- `npx tsc --noEmit`로 타입 안정성 확인 (이번 세션에서 import 라우트 수정 시
  사용한 컨벤션과 동일)
- 브라우저 dev 서버에서 수동 검증: 캐릭터 1명 / 2명 이상으로 게임모드 대화 생성,
  세 슬래시 커맨드 모두 실행, 여러 턴에 걸쳐 점수 델타가 누적/클램핑되는지 확인,
  재생성 및 메시지 삭제 시 롤백으로 점수가 복원되는지 확인, `!엔딩`이 캐릭터별
  개별 엔딩을 생성하는지 확인
- 새 자동화 테스트 인프라는 추가하지 않음 — `storyEval.ts`를 검증하는 자동
  테스트가 현재도 없으므로 기존 컨벤션을 따름
