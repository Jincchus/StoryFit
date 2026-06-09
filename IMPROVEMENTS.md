# StoryFit 전체 코드 리뷰 & 개선 로드맵

> 작성: 2026-06-01 · 대상: `apps/web` (Next.js 14 + Prisma + Gemini)
> 우선순위 기준 — **주력(게임형 스토리 롤플) / 적은 토큰으로 기억 유지 / 편의성** 3가지에 가중치.
>
> 표기: 🔴 P0(즉시·고효과) · 🟠 P1(높음) · 🟡 P2(중간) · ⚪ P3(여유 될 때)
> 분류: `[기억/토큰]` `[스토리]` `[UX]` `[성능]` `[리팩터링]` `[안정성]`

---

## 0. 한눈에 보는 핵심 진단

| # | 문제 | 영향 | 우선순위 |
|---|------|------|----------|
| 1 | DB에 인덱스가 **하나도 없음** (벡터 HNSW만 수동 생성) | 대화 길어질수록 모든 쿼리 느려짐 | 🔴 |
| 2 | 컨텍스트를 **토큰 예산이 아닌 "최근 15개 메시지 고정"**으로 자름 | 긴 응답 누적 시 토큰 폭증 + 잘림 | 🔴 |
| 3 | 스토리 1턴에 **AI 호출 2~4회** (본문+재작성+스탯+인벤) | 비용·지연 2~4배 | 🔴 |
| 4 | `coreMemory`/`statusTimeline`이 **100% 수동 입력** | "기억 유지"의 핵심인데 자동화 안 됨 | 🟠 |
| 5 | 단일 캐릭터 채팅이 **진짜 스트리밍이 아니라 1.5초 폴링** | 응답이 뚝뚝 끊겨 보임 | 🟠 |
| 6 | 시스템 프롬프트 **캐싱 미사용** (매 턴 전체 재전송) | 입력 토큰 낭비 | 🟠 |
| 7 | 채팅 페이지 **1362줄 단일 컴포넌트, 상태 40개+** | 유지보수·버그 추적 난이도 | 🟡 |
| 8 | 유틸 호출(요약/스탯/인벤)에 **thinking 끄지 않음** | 백그라운드 호출 느려짐 | 🟡 |

---

## 🔴 P0 — 즉시 (효과 대비 비용 최고)

### P0-1. `[성능]` Prisma 스키마에 인덱스 추가
**현재:** `schema.prisma` 전체에 `@@index`가 **0개**. 매 요청마다 도는 쿼리들:
- `Message`: `where conversationId + isSelected + isStreaming, orderBy createdAt` (채팅 로드·요약·컨텍스트마다)
- `Memory`: `where conversationId, orderBy createdAt`
- `Lorebook`: `where conversationId / scope+scopeId`
- `Conversation`: `where userId` (목록)

데이터가 수천 건 쌓이면 전부 풀스캔. **지금은 빠르지만 사용자·대화가 늘면 급격히 느려짐.**

**조치:**
```prisma
model Message {
  // ...
  @@index([conversationId, createdAt])
  @@index([conversationId, isSelected, isStreaming])
}
model Memory       { @@index([conversationId, createdAt]) }
model Lorebook     { @@index([conversationId]) @@index([scope, scopeId]) }
model Conversation { @@index([userId, updatedAt]) }
model AiErrorLog   { @@index([createdAt]) }
```
`start.sh`가 부팅 시 `prisma db push`를 돌리므로 자동 반영됨. **비용 거의 0, 효과 큼.**

---

### P0-2. `[기억/토큰]` 컨텍스트를 "메시지 개수"가 아니라 "토큰 예산"으로 자르기
**현재:** `chat/route.ts:108` `const recentMsgs = conv.messages.slice(-15)`
- 항상 최근 15개를 **전문 그대로** 전송.
- maxOutputTokens를 8K→16K로 올린 지금, 한 응답이 한글 8000자(≈16K토큰)까지 가능 → 최근 15개면 **단일 요청 입력이 수십만 토큰**까지 폭증 가능.
- "적은 토큰으로 기억 유지"라는 목표와 정면 충돌.

**조치 (단계적):**
1. **토큰 예산 기반 슬라이딩 윈도우**: 최근 메시지를 뒤에서부터 누적 토큰(`approxTokens` 재사용)이 예산(예: 6~8K)을 넘기 직전까지만 포함. 개수가 아니라 예산으로 컷.
2. 윈도우에서 밀려난 구간은 **요약(Memory)으로 대체** — 이미 `retrieveRelevantMemories`가 있으니, "최근 raw 윈도우 + 그 이전은 요약" 경계를 명확히 맞춤.
3. 현재 요약(매 10개)과 raw 윈도우(최근 15개)가 **겹치는 구간**(최근 10~15개가 요약+원문 중복 전송)을 정리.

→ 토큰 사용량을 **응답 길이와 무관하게 상한 고정**할 수 있어 비용·속도·잘림 모두 개선.

---

### P0-3. `[스토리][성능]` 스탯·인벤 평가를 1콜로 합치고, 변화 없을 땐 스킵
**현재 (주력 모드인 스토리에서 매 턴):**
- `chat/route.ts:228-233` → `triggerStatsEvaluation`(별도 Gemini 콜) + `triggerInventoryEvaluation`(또 별도 콜)
- 거기에 `needsResponseRevision` 걸리면 **재작성 콜**까지.
- 즉 **스토리 1턴 = 본문 스트림 + (재작성) + 스탯 + 인벤 = 최대 4회 호출.**

**조치:**
1. **스탯+인벤을 단일 `generateText` 호출로 병합** — 하나의 JSON(`{stats:{...}, inventory:{add,remove}}`)으로 반환받아 파싱. 콜 수 절반.
2. **변화 가능성 휴리스틱 게이트**: 유저 입력/AI 응답에 아이템·수치 관련 키워드가 전혀 없으면 평가 콜 자체를 스킵(또는 N턴마다).
3. (중기) 본문 생성 시 구조화 출력으로 델타를 같이 받아 **별도 콜 제거** 검토.

→ 주력 모드의 비용·지연을 직접적으로 절반 이하로.

---

## 🟠 P1 — 높음

### P1-1. `[기억/토큰][스토리]` `statusTimeline` 자동 유지 (씬 상태 트래킹)
**현재:** `coreMemory`, `statusTimeline` 둘 다 **사용자가 패널에서 손으로 입력**(`page.tsx:523-531`). "기억 유지"의 핵심 필드인데 자동화가 없음.

**제안:** 매 턴(또는 N턴) **가벼운 유틸 콜**로 `statusTimeline`을 자동 갱신 — 현재 위치/시간/동석 인물/직전 상황을 **3~5줄로 압축**해 덮어씀(누적 아님). 
- 이게 있으면 "최근 15개 원문"에 의존하지 않고도 **적은 토큰으로 연속성**을 유지 → P0-2와 시너지.
- 비용은 짧은 출력이라 작음. P0-3에서 스탯/인벤과 **같은 병합 콜에 끼워넣으면 추가 콜 0**.

### P1-2. `[UX][성능]` 단일 캐릭터 채팅도 진짜 SSE 스트리밍으로
**현재:** `chat/route.ts`는 202 + messageId 반환 → `conversationStream.ts`가 **1.5초마다 폴링**, 서버는 2초마다 DB flush. 체감상 응답이 ~2초 단위로 뭉텅뭉텅 나타남(`tikiTaka`만 진짜 SSE).
- bg+폴링 구조의 장점(중간 이탈/재접속에도 부분 저장 유지)은 분명함 → CLAUDE.md의 "partial stream save" 규칙과 연결.

**제안 (택1):**
- (가벼움) flush 간격 2초→0.5초 + 폴링 1.5초→0.7초로 체감 개선.
- (제대로) SSE로 토큰 즉시 전송 + 동시에 DB에 부분 저장(현 안정성 유지). 재접속 시엔 폴링 폴백.

### P1-3. `[기억/토큰][성능]` Gemini 컨텍스트 캐싱 적용
**현재:** `gemini.ts`가 매 턴 `systemInstruction`(베이스룰+캐릭터+시나리오+로어북+요약 = 수천 토큰)을 **전부 재전송**. 대화 안에서 이 앞부분은 거의 불변.

**제안:** Gemini **explicit context caching**(또는 최소한 안정적 프리픽스 유지로 implicit 캐시 히트 유도)으로 정적 프리픽스 입력 토큰 비용 절감. "적은 토큰" 목표에 직접 기여. 단, 캐시 최소 토큰/TTL 조건 확인 필요.

### P1-4. `[스토리]` 선택지 파싱·검증의 취약성 완화
**현재:** 스토리 선택지는 AI가 `---` 구분선 + `1. 2. 3. 4.` 형식을 **정확히** 지켜야 동작. 파싱은 `parseStoryChoices`(클라, `page.tsx:38`), 검증은 `responseControl.ts`의 정규식 + 위반 시 **전체 재작성 콜**.
- 형식이 조금만 어긋나도 재작성(비용↑) 또는 선택지 누락.

**제안:**
- 본문/선택지를 구분선 대신 **구조화 출력(JSON 또는 명시적 태그 `<choices>`)**으로 받으면 파싱·검증이 견고해지고 재작성 빈도↓.
- 최소한 `parseStoryChoices`가 `---` 변형(`***`, `===`, 공백)과 `①②③④`도 허용하도록 보강(서버 `getChoiceBlock`은 이미 일부 지원 → 클라와 정규식 통일 필요).

### P1-5. `[안정성]` 광범위한 silent catch 제거
**현재:** `page.tsx`·라우트 곳곳에 `.catch(() => {})`, `catch {}`. 로어북 저장 실패, 메모리 삭제 실패, 패치 실패가 **사용자에게 아무 피드백 없이** 사라짐.
**제안:** 사용자 액션 실패는 Toast로 노출, 서버는 `logAiError` 패턴처럼 최소 로깅. "편의성" 체감에 직결.

### P1-6. `[성능]` 메시지 로딩 페이지네이션
**현재:** `messages/route.ts` GET이 대화의 **모든 메시지**를 매번 로드 + 클라가 전부 렌더(`page.tsx`). 수백 턴 스토리에서 초기 로딩·메모리 부담.
**제안:** 최근 N개만 로드 + "위로 더 보기" 무한스크롤. (AI 컨텍스트는 P0-2가 따로 처리하므로 독립적으로 적용 가능.)

---

## 🟡 P2 — 중간

### P2-1. `[성능]` 유틸 호출에 `thinkingBudget: 0` 명시
`gemini.ts:generateText`는 `generationConfig`가 아예 없음 → gemini-2.5-flash **기본 thinking이 켜진 채** 요약/스탯/인벤/시나리오 생성이 돌 수 있음(느림·비용↑). 유틸 콜엔 `thinkingConfig:{thinkingBudget:0}` + 적정 `maxOutputTokens` 명시 권장.

### P2-2. `[리팩터링]` `conversations/[id]/page.tsx` 분해 (1362줄 / 상태 40+)
커스텀 훅으로 분리: `useChatStream`, `useMemoryPanel`, `useLorebook`, `useBranches`, `useSpeech(STT/TTS)`. 렌더는 `<Composer>`, `<MessageList>`, `<SidePanel>`, `<ChoiceList>` 등 서브컴포넌트로. 버그 추적·기능 추가 속도 크게 향상.

### P2-3. `[스토리]` 스탯/인벤 평가 입력 truncation 재검토
`statsEval`/`inventoryEval`이 `aiMsg.slice(0, 400)`만 사용 → 긴 응답의 후반 사건(아이템 획득 등)을 놓침. 핵심 이벤트 위주 요약 입력 또는 상한 상향.

### P2-4. `[기억/토큰]` 로어북 매칭 정교화
`matchLorebook`(`systemPrompt.ts:249`)이 per-entry `scanDepth`를 무시하고 전역 5 사용 + 단순 `includes`(부분 문자열 오매칭 가능). 엔트리별 `scanDepth` 반영 + 단어 경계/정규화 고려.

### P2-5. `[UX]` 컨텍스트 사용량/토큰 표시
입력 토큰·컨텍스트 점유율을 채팅 화면에 가볍게 표시(이미 `inputTokens/outputTokens` 저장 중). "기억 유지"가 핵심인 앱에서 사용자가 컨텍스트 상태를 체감하게 함.

### P2-6. `[UX]` 선택지/입력 편의 기능
- **확인됨**(`page.tsx:806,813`): 선택지는 마지막 AI 메시지에서만 뜨고 **클릭 즉시 전송** — 수정 단계 없음. → 선택지를 composer에 **채워 넣고 수정 후 전송**하는 옵션 추가 권장(특히 4번 "씬 진행" 선택지를 다듬고 싶을 때).
- "이어쓰기/계속" 버튼(짧게 끊긴 응답 연장), 마지막 유저 입력 **재전송** 단축키.
- STT/TTS는 이미 구현됨(브라우저 `SpeechRecognition`/`speechSynthesis`) — 기능 추가가 아니라 음성 선택·속도 등 다듬기 영역.

### P2-7. `[안정성]` 문서-코드 불일치 정리 (`parentId`/branch)
CLAUDE.md는 "Message.parentId v1 always null, 브랜치 v2"라지만 코드엔 branch 생성/전환/sibling 계산이 **이미 동작**. 문서 또는 코드 중 하나로 정합. 혼선 방지.

---

## ⚪ P3 — 여유 될 때

- `[성능]` `retrieveRelevantMemories`가 매 턴 쿼리 임베딩 1콜 → 짧은 입력은 캐시/스킵 검토.
- `[리팩터링]` `statsEval`/`inventoryEval`의 `extractJson`·`editDistance` 중복 → 공용 util로.
- `[UX]` 모바일(`apps/mobile`)과 웹 동작 일치 점검(이번 리뷰 범위 밖).
- `[안정성]` `generateText` 응답이 빈 JSON/깨진 JSON일 때 재시도 1회.
- `[UX]` 대화 목록 검색/필터/태그, 캐릭터별 정렬.
- `[기억/토큰]` 요약 품질: 중요 사건 가중(관계 변화·결정)·고정 10개 청크 대신 의미 단위 청킹 검토.

---

## 추천 진행 순서

1. **P0-1 (인덱스)** — 5분, 위험 0, 즉시 머지.
2. **P0-3 (스탯+인벤 병합/게이트)** + **P1-1 (statusTimeline 자동화)** — 같은 병합 콜에 넣으면 주력 모드 비용/지연/기억 동시 개선.
3. **P0-2 (토큰 예산 윈도우)** — 기억·비용 핵심.
4. **P1-3 (캐싱)** , **P1-2 (SSE)** — 비용·체감 속도.
5. **P2-2 (페이지 분해)** — 위 작업들 끝난 뒤 안정화 단계에서.

> 각 항목은 독립적으로 머지 가능하도록 작게 쪼갤 수 있음. 착수할 항목 골라주면 설계→구현으로 들어가겠음.
