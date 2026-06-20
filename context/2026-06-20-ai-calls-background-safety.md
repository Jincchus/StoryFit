# AI 호출 백그라운드 안전성 — 감사 & 수정

> 작성일: 2026-06-20
> 목적: 채팅 외 모든 AI 호출이 모바일 WebView 백그라운드 전환에도 정상 동작하는지 확인·보강

## 서버 측 결론: 이미 안전
- 비채팅 AI 라우트는 전부 `lib/ai/gemini.ts`의 **`generateText`** 사용 → **클라이언트 요청 signal에 묶지 않음**.
  따라서 앱이 백그라운드로 가서 fetch가 끊겨도 **서버는 AI 호출을 끝까지 완료하고 DB write도 실행**된다.
  (앱은 standalone Node `server.js`로 구동 — 핸들러가 클라 disconnect로 강제 종료되지 않음.)
- 채팅(`chat/regenerate/continue/assistant`)은 별도로 **detached 패턴**: 서버 자체 AbortController(5분)
  + `streamBroker` + DB 선저장 → 클라 연결과 완전 분리. (`chat/route.ts`)

## 라우트별 분류
| 라우트 | 결과 저장? | 백그라운드 |
|---|---|---|
| `characters/[id]/openings/generate` | ✅ character.openingMessages 저장 | 안전(저장) |
| `lorebooks/import` | ✅ lorebook.create | 안전(저장) |
| `collections/[id]/translate` (chub) | ✅ character+chubMeta 저장 | 안전(저장) |
| memorySummarization/recap/coreMemory (lib) | ✅ 서버 트리거(fire-and-forget) | 안전 |
| `conversations/[id]/recap` | ❌ 반환만(일회성 패널) | 재요청-안전(손실 아님) |
| `conversations/[id]/suggestions` | ❌ 반환만(추천 답변 칩) | 재요청-안전 |
| `conversations/generate-scenario` | ❌ 폼 초안 | 재요청-안전 |
| `characters/generate` | ❌ 폼 초안 | 재요청-안전 |

→ 저장형은 데이터 손실 없음. 미저장형은 폼/패널용 일회성이라 다시 누르면 됨(저장된 데이터 손실 아님).

## 진짜 공백: 클라이언트 UX
백그라운드 중 in-flight fetch가 멈추거나 에러나면, **서버는 저장했는데 화면이 안 갱신**되거나
스피너가 멈춘 채 남을 수 있다. 채팅은 broker 폴링 + `visibilitychange→loadConv`로 복구하지만
다른 화면엔 그 복구가 없었다.

## 수정
- **`lib/useRefetchOnForeground.ts`** 신설: `visibilitychange`(visible)·`focus` 시 콜백 실행 훅.
- 적용(포그라운드 복귀 시 재조회 + 멈춘 스피너 정리):
  - chub 상세: 컬렉션 재조회 + `translating` 해제. 추가로 **번역 POST가 드롭돼도** 재조회해
    `chubMeta.activeLang` 변화로 성공 감지 → 복구.
  - melting 상세: 컬렉션 재조회 + `generatingOpening` 해제(AI 이어쓰기 결과 반영).
  - `useLorebook`: 로어북 목록 재조회 + `lorebookImporting` 해제(AI 로어북 가져오기 반영).
- 채팅 페이지는 기존 `visibilitychange→loadConv`로 이미 커버.

## 남긴 것(의도)
- recap/suggestions/시나리오·캐릭터 생성: 미저장 일회성. 백그라운드 시 결과만 못 받을 뿐
  저장 데이터 손실 없음 → 재요청으로 충분. 별도 영속화는 과함(스키마 추가 필요)이라 보류.
