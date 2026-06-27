# 센터 페르소나·캐릭터 등록 흐름 통일 설계

작성일: 2026-06-27
대상: 9개 센터 카드 상세 페이지 + 공용 모듈/모달 + systemPrompt + 일부 API

## 배경

9개 센터(whif·zeta·melting·tikita·chub·rofan·loveydovey·babechat·tingle)의 카드 상세
페이지가 "채팅 시작 → 페르소나 선택" 로직을 거의 동일하게 복붙하고 있으나 세부가 어긋나 있다.

조사 결과(현황):
- **호감도 스탯·추천답변 기본값 불일치**: melting·rofan·babechat·loveydovey·chub는 둘 다 ON,
  zeta·whif·tingle은 둘 다 OFF, tikita는 추천답변만 ON.
- **페르소나 후보 불일치**: zeta·whif만 기존 캐릭터를 페르소나 후보로 제시(`candidates`),
  나머지는 `candidates={[]}`라 새 페르소나 입력만 가능.
- **캐릭터 등록(단일→멀티) 기능은 zeta만** 보유(`+ 직접 캐릭터 등록`).
- **페르소나 placeholder 치환**: 어제 커밋 `496b60a`가 페르소나 카드의 `{{char}}/{{user}}`를
  flip(뒤집기) 처리. 단 항상 flip 고정이라 사용자가 방향을 고를 수 없음.

## 목표 (사용자 확정)

1. **기본값 통일** — 전 센터 호감도 스탯(50) + 추천답변을 기본 ON.
2. **페르소나 모달 통일** — 모든 센터에서 선택(기존 캐릭터)+입력(새 페르소나) 둘 다 가능.
   - 후보 = (a) 현재 카드가 멀티면 그 안의 다른 캐릭터(멀티 동료) + (b) 내 standalone 카드.
3. **캐릭터 등록(단일→멀티)** — 전 센터 카드 상세에 zeta식 등록 버튼 추가.
4. **페르소나 치환 토글** — 페르소나 카드 설정의 placeholder를 flip할지 사용자가 선택.

## 핵심 데이터 정의

- **standalone 카드** = `collectionId === null`인 내 캐릭터. 복제(`duplicate` 라우트가
  `collectionId:null`로 생성)·직접 생성(collectionId 미지정) 카드가 해당. 외부 센터
  세계관/서사/테마에 매핑되지 않은 카드.
- **멀티 동료** = 현재 카드(컬렉션)의 `characters` 중 AI 캐릭터로 선택되지 않은 나머지.

## 설계

### A. 공용 모듈 추출 (중복 제거)

기존 9개 페이지의 `startChat`/`handlePersonaSelect`/모달 호출을 공용 단위로 추출한다.

- **`lib/centerChat.ts`**
  - `buildPersonaCandidates({ collectionChars, aiCharIds, standaloneCards }): Candidate[]`
    — 멀티 동료 + standalone 병합, `aiCharIds`·중복 제거. 각 항목 `{id,name,gender,avatarUrl}`.
  - `createCenterChat({ col, aiCharIds, persona, opening, flipPlaceholders, extras }): Promise<convId>`
    — 새 페르소나면 먼저 `POST /api/characters`(`collectionId: col.id`)로 생성 후 그 id 사용.
      `POST /api/conversations`에 **통일 기본값**(`statsEnabled:true`,
      `statsConfig:[{name:'호감도',value:50,min:0,max:100}]`, `suggestRepliesEnabled:true`) +
      `personaCharacterId` + `personaFlipPlaceholders` + 센터별 `extras`(zeta 시나리오·도입부 등) 병합.
- **`components/ui/ChatModeModal.tsx`** — zeta 인라인 "전체와 대화 / 각 캐릭터 1:1" 모달을
  컴포넌트로 추출. `col.characters.length > 1`일 때 표시, 선택 결과로 `aiCharIds` 결정.
- **`WhifPersonaModal`** 재사용. 변경점: 병합된 candidates 수용(이미 지원) + **치환 토글 1개 추가**.

각 센터 페이지는 위 모듈을 호출하는 얇은 wrapper가 된다(센터별 extras·테마 칩만 차이).

### B. 통합 페르소나 모달

- 모든 센터가 `buildPersonaCandidates(...)` 결과를 `WhifPersonaModal`에 전달.
- standalone 후보 조회: **신규 `GET /api/characters?unassigned=true`** — `creatorId=userId AND
  collectionId=null` 필터, 목록 select(id·name·gender·avatarUrl)만. (전체 `/api/characters`를
  불러와 클라이언트 필터하지 않고 서버에서 좁힘.)
- 모달은 기존 "기존 캐릭터/새 페르소나" 2탭 유지. 후보가 있으면 "기존" 탭 노출.

### C. 캐릭터 등록 (단일→멀티)

- 전 센터 카드 상세에 `+ 캐릭터 등록` 버튼 → `/characters/new?is<Center>=true&collectionId=${col.id}`.
  (`<Center>` 파라미터는 각 센터가 이미 보유.)
- 등록 후 컬렉션이 멀티가 되면 채팅 시작 시 `ChatModeModal`이 역할 선택을 받는다.
  사용자 의도: 등록 캐릭터=AI, 기존 캐릭터=페르소나(후보로 자동 노출).

### D. 기본값 통일

`createCenterChat`이 모든 센터에 호감도50+추천답변 ON을 적용하므로, zeta·whif·tingle·tikita도
자동으로 통일된다. 멀티(multiStory)에도 동일 적용(스탯은 대화방 단위).

### E. 페르소나 치환 토글

- **스키마**: `Conversation.personaFlipPlaceholders Boolean @default(true)` 추가(`prisma db push`).
- **systemPrompt** (`lib/systemPrompt.ts`) 페르소나 블록 — 단일/멀티 양쪽 분기 모두:
  - flip ON(기본): `replacePlaceholders(info, <AI캐릭터명>, <페르소나명>)`
    → `{{char}}→페르소나`, `{{user}}→AI캐릭터` (현재 동작)
  - flip OFF: `replacePlaceholders(info, <페르소나명>, <AI캐릭터명>)`
    → `{{char}}→AI캐릭터`, `{{user}}→페르소나`
  - 멀티는 `<AI캐릭터명>` 자리에 `characters.map(c=>c.name)` 배열/조인을 그대로 사용.
- **조작 위치**: 페르소나 모달에서만(대화 생성 시 1회). 값은 `Conversation`에 저장되어 매 턴 반영됨.
  **채팅방 내 토글은 두지 않는다**(설정 혼선 방지) → SidePanel·PATCH 변경 없음.

#### 의미 예시
원본 카드 `민수`(`{{char}}는 {{user}}를 짝사랑하는 소꿉친구`) → `지영` 등록, 지영=AI·민수=페르소나:

| 토글 | 결과 | 의미 |
|---|---|---|
| ON(flip, 기본) | "민수는 지영을 짝사랑하는 소꿉친구" | 원본 설정의 '나'=페르소나(민수) |
| OFF | "지영은 민수를 짝사랑하는 소꿉친구" | 설정이 새 AI 캐릭터(지영) 기준 |

## 영향 파일

- 신규: `lib/centerChat.ts`, `components/ui/ChatModeModal.tsx`
- 수정: `prisma/schema.prisma`, `lib/systemPrompt.ts`(+ `buildMultiStorySystemPrompt`),
  `lib/chatPipeline.ts`(systemPrompt 빌더 **유일** 호출부 — `conversation.personaFlipPlaceholders`를
  읽어 빌더에 전달. 따라서 chat/regenerate/continue/recap 라우트 개별 수정 불필요),
  `components/ui/WhifPersonaModal.tsx`, `app/api/characters/route.ts`(unassigned 필터),
  `app/api/conversations/route.ts`(personaFlipPlaceholders 수용), 9개 센터 상세 페이지,
  `app/(main)/guide/page.tsx`(기능 가이드 동기화).

## 알려진 한계

- multiStory + flip ON에서 페르소나 카드의 `{{user}}`는 *AI 캐릭터 이름들을 쉼표로 나열*한
  값으로 치환된다(기존 동작). 어색하면 OFF 토글로 회피 가능.
- 토글은 기존 캐릭터(자체 additionalInfo 보유)를 페르소나로 쓸 때 의미가 있다. 새로 입력한
  페르소나는 placeholder가 거의 없어 토글 영향이 미미(기본 ON 무해).

## 범위 외 (YAGNI)

- 채팅방 내 치환 토글 변경(PATCH) — 명시적 제외.
- 세계관/캐릭터 구분 필터, 페르소나 카드의 `{{char1}}` 인덱스 정교화 — 후속 과제.

## 시스템 프롬프트 순서 불변 확인

본 변경은 페르소나 블록의 **치환 방식**과 대화 생성 기본값만 바꾸며, CLAUDE.md의 고정
조립 순서(UserPersona 위치 포함)는 변경하지 않는다.
