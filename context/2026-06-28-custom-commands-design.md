# 사용자 정의 AI 커맨드 (`!에타` 류) + 커맨드 응답 마크다운 렌더링

작성일: 2026-06-28

## 배경 / 목적

현재 채팅 커맨드(`lib`·`app/api/conversations/[id]/chat/route.ts`의 "확장형 커맨드 엔진", `chatShared.ts`의 `COMMANDS`)는
`!상태창`·`!스탯`·`!인벤토리`·`!상황`·`!도움말`처럼 **서버가 저장값을 AI 호출 없이 즉시 출력**하는 무-AI 조회 커맨드다.

사용자는 그 외에 **채팅 중 설정창에서 직접 커맨드를 만들어** AI에게 정해진 지시를 보내고 싶어한다.
예: `!에타` = "현재 상황을 확인해 에브리타임 게시글 형식으로 작성하라". 한 번 만들면 `!에타`만 쳐서 계속 같은 형식의
응답을 받는다. 즉 **AI 프롬프트 매크로**. 또한 그 응답을 **마크다운으로 렌더**해 에브리타임 게시판·채팅창 같은
모양을 내고 싶어한다(현재 메시지는 노벨 스타일 렌더 — `components/ui/NovelText.tsx`).

## 합의된 결정 (브레인스토밍)

- **범위**: 계정별(글로벌). 한 번 만들면 모든 채팅·캐릭터에서 사용.
- **맥락 처리**: 커맨드 응답은 일반 assistant 메시지로 저장(이후 맥락 포함, 분기/재생성 그대로).
- **인자**: `!에타 [추가텍스트]` — 뒤 텍스트를 추가 지시로 이어붙임(option 2).
- **마크다운 렌더 범위**: 커스텀 + 빌트인 커맨드 응답 **모두**. (빌트인의 `###`·`**`도 제대로 렌더되어 개선)
- **저장/실행 방식**: A안 — 계정별 DB + 서버 실행(채팅 라우트가 `!이름` 감지·지시문 주입). 동기화·깔끔한 트랜스크립트.

## 설계

### 1. 데이터 모델 (Prisma)

- 신규 모델 **`UserCommand`**: `id, userId, name, instruction, description?, createdAt`.
  - `name`: `!` 제외한 이름(예: `에타`). `@@unique([userId, name])`.
  - `instruction`: AI에 주입할 지시문(유저 작성).
  - `description?`: 자동완성 메뉴 설명(선택).
- **`Message.commandName String?`** 컬럼 추가: 커맨드로 생성/응답된 메시지에 커맨드 이름을 기록.
  렌더러가 이 값의 유무로 **마크다운 vs 노벨 스타일**을 분기. (커스텀·빌트인 모두 세팅)
- 마이그레이션 2건(모델 추가, 컬럼 추가).

### 2. API — `/api/commands`

- `GET /api/commands` — 로그인 유저의 커맨드 목록.
- `POST /api/commands` — 생성 `{ name, instruction, description? }`.
- `PATCH /api/commands/[id]` — 수정.
- `DELETE /api/commands/[id]` — 삭제.
- 전부 `authenticate(req)`로 유저 스코프. 다른 유저 커맨드 접근 금지.
- **검증**: name은 trim 후 공백·`!`·특수문자 제한(한글/영문/숫자/_ 정도), 길이 상한, 빌트인 예약어
  (`상태창/정보/status/스탯/인벤토리/소지품/인벤/inventory/상황/씬/scene/타임라인/도움말/명령어/help`)·중복 금지,
  instruction 비어있으면 거부.

### 3. 실행 흐름 (채팅 라우트 커맨드 엔진)

메시지 `!이름 [추가텍스트]` 전송 시 서버:

1. `lib/commands.ts`의 순수 파서로 `{ name, args }` 분리(첫 토큰=이름, 나머지=args).
2. **빌트인 이름**이면 → 상태 데이터 유무로 분기(아래 §3a). 저장되는 reply 메시지엔 항상 `commandName` 세팅(마크다운 렌더).
3. 빌트인이 아니면 → 유저 `UserCommand` 조회.
   - **있으면**: 이번 턴을 **일반 AI 생성**으로 처리.
     - 유저 메시지는 `!이름 [추가텍스트]` 원문 그대로 저장(표시용).
     - AI 생성 시 **이 턴 한정 지시 블록**을 주입:
       `[사용자 커맨드: {이름}]\n{instruction}\n{args가 있으면: 추가 지시: {args}}\n\n응답은 마크다운 형식으로 작성하라.`
       (이번 생성 요청에 한해 시스템 프롬프트 끝(closingRules 뒤)에 임시 지시 블록으로 덧붙인다 — DB 영속 저장 X, 고정 순서 자체는 유지)
     - 기존 스트리밍·부분저장·분기/재생성 파이프라인 **그대로 재사용**.
     - 생성된 assistant 메시지에 `commandName={이름}` 세팅(마크다운 렌더 + UI 라벨 가능).
   - **없으면**: 기존 "알 수 없는 명령어" 응답(이것도 `commandName` 세팅해 일관 렌더).

### 3a. 빌트인 상태 커맨드의 AI 폴백 (값 없으면 AI가 상태창 생성)

대상: `!상태창`·`!스탯`·`!인벤토리`·`!상황` (도움말 제외).

- **관련 데이터가 있으면**: 기존처럼 서버가 저장값을 무-AI로 즉시 출력(+ `commandName` 세팅 → 마크다운 렌더).
- **비어있으면**: "값 없음" 안내 대신 **AI 생성으로 전환**.
  - §3의 커스텀 경로와 동일하게 이번 턴 한정 지시 블록을 주입:
    `[시스템 커맨드: {이름}] 현재 상황(직전 대화·타임라인·기존 스탯/인벤토리)을 근거로 {상태창=스탯+호감도+소지품+상황 / 스탯 / 인벤토리 / 현재 상황}을 합리적으로 추론해 마크다운 형식의 상태창으로 작성하라.`
  - 일반 AI 생성 파이프라인 재사용, assistant 메시지에 `commandName` 세팅, 마크다운 렌더.
- **비어있음 판정**: 스탯 배열 / 인벤토리 배열 / `statusTimeline` 문자열이 비었는지로 각각 결정.
  통합 `!상태창`은 셋이 모두 비었을 때만 AI 폴백(일부라도 있으면 기존 즉시 출력).
- 주의: 폴백은 AI를 태우므로 비용 발생 — 데이터가 있으면 항상 무-AI 즉시 출력이 우선.

### 4. 렌더링

- 의존성 추가: **`react-markdown` + `remark-gfm`**(표·목록·체크박스·구분선 등). 원시 HTML 미렌더(기본 안전, XSS 방지).
- 신규 컴포넌트 **`components/ui/MarkdownText.tsx`**: react-markdown 래퍼(앱 톤에 맞춘 최소 스타일).
- 메시지 렌더 분기(`components/ui/MessageBlocks.tsx` 또는 `MessageList.tsx`에서 NovelText를 쓰는 지점):
  `message.commandName`이 있으면 `<MarkdownText>`, 없으면 기존 `<NovelText>`(노벨 스타일).
- 선택: 커맨드 메시지 상단에 작은 라벨/배지(예: `📋 에타`) — UI 다듬기 단계에서.

### 5. UI (설정)

- **SidePanel "내 커맨드" 섹션**: 목록(이름·설명) + 추가(이름·지시문·설명 입력) + 수정/삭제.
- **자동완성 메뉴(`CommandMenu`)**: 빌트인 `COMMANDS` + 유저 커맨드(fetch)를 합쳐 `!` 입력 시 표시.
  커스텀은 설명과 함께, 빌트인과 구분되는 가벼운 표식.
- `app/(main)/conversations/[id]/page.tsx`의 `filteredCommands`가 합쳐진 목록을 사용.

### 6. 검증 / 에러

- CRUD 검증은 §2. 커맨드 실행 중 AI 실패 시 기존 부분저장/에러 처리 그대로.
- 빈 instruction의 커맨드는 생성 단계에서 차단되므로 실행 시 항상 유효.

### 7. 테스트

- **순수 로직 vitest TDD** (`lib/commands.ts` + `lib/commands.test.ts`):
  - `parseCommand(text)` → `{ name, args }` (첫 토큰/나머지 분리, 앞 `!` 처리, 공백 트림).
  - `isBuiltinCommand(name)` → 예약어 판별.
  - `composeCommandDirective(instruction, args)` → 주입 지시 블록 문자열(마크다운 출력 지시 포함).
- CRUD 라우트·렌더 컴포넌트·SidePanel·자동완성: `npm run build`(타입체크) + 수동 검증.

## 영향 범위 / 파일

- 수정: `prisma/schema.prisma` (UserCommand 모델, Message.commandName) + 마이그레이션.
- 신규: `lib/commands.ts`(+test), `app/api/commands/route.ts`, `app/api/commands/[id]/route.ts`,
  `components/ui/MarkdownText.tsx`.
- 수정: `app/api/conversations/[id]/chat/route.ts`(커스텀 조회·지시 주입·commandName 세팅),
  메시지 렌더 분기(`components/ui/MessageBlocks.tsx`/`MessageList.tsx`),
  `app/(main)/conversations/[id]/_components/SidePanel.tsx`(내 커맨드 CRUD),
  `app/(main)/conversations/[id]/_lib/chatShared.ts`·`page.tsx`(자동완성 병합).
- 의존성: `react-markdown`, `remark-gfm` 추가.
- **가이드 동기화**(CLAUDE.md 규칙): 사용자 체감 기능이므로 `app/(main)/guide/page.tsx` `FEATURE_SECTIONS`에 추가.

## 비목표 (YAGNI)

- 인자 고급 문법(이름 있는 파라미터, 치환 토큰)은 제외 — 뒤 텍스트 단순 이어붙이기만.
- 유저 간 커맨드 공유/마켓, 캐릭터별 스코프, 커맨드 아이콘 커스터마이즈.
- 빌트인 커맨드 엔진의 동작 자체 변경(마크다운 렌더 플래그만 추가, 출력 로직은 유지).

## 검증 기준

- 설정창에서 `!에타` 커맨드 생성 후, 채팅에서 `!에타`(또는 `!에타 추가지시`) 입력 시 지시문 기반 AI 응답이 생성된다.
- 그 응답이 **마크다운으로 렌더**되어 게시판/채팅창 느낌이 난다.
- 빌트인 커맨드(`!상태창`·`!도움말`) 응답도 마크다운으로 깔끔히 렌더된다.
- 상태값이 비어있을 때 `!상태창`·`!스탯`·`!인벤토리`·`!상황`이 **AI가 생성한 마크다운 상태창**을 보여준다(값이 있으면 기존 무-AI 즉시 출력).
- 커맨드는 모든 대화에서 재사용되고(글로벌), 자동완성에 뜬다.
- 일반 메시지는 기존 노벨 스타일 그대로(회귀 없음).
