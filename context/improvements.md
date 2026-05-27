# StoryFit 개선점 목록

> 2026-05-24 전체 프로젝트 스캔 결과. 우선순위 순 정렬.

---

## 🔴 Critical — 보안

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| S-1 | ✅ 완료 | `app/api/conversations/[id]/route.ts` | PATCH·DELETE 시 userId 소유권 검증 없음 → 타 유저 대화 수정/삭제 가능 | `updateMany/deleteMany + where userId` 적용 |
| S-2 | ✅ 해당없음 | `app/api/characters/[id]/route.ts` | 캐릭터 PATCH·DELETE 소유권 검증 누락 | 이미 creatorId 검증 로직 존재 |
| S-3 | ✅ 완료 | `app/api/conversations/[id]/messages·memories·chat·regenerate` | 메시지·메모리·채팅 조작 시 conversation 소유권 미검증 | 모든 엔드포인트에 conv.userId 검증 추가 |
| S-4 | ✅ 완료 | `app/api/characters/route.ts` | avatarUrl URL 형식·이름 길이·태그 수 무검증 | 최대 길이 제한 + URL 형식 검증 |
| S-5 | ✅ 해당없음 | `lib/authClient.ts` | localStorage에 실제 토큰이 아닌 플래그('1')만 저장. 실제 JWT는 httpOnly 쿠키에 있어 안전 | 추가 조치 불필요 |

---

## 🔴 Critical — 로직 버그

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| L-1 | ✅ 완료 | `lib/rateLimit.ts` | userTimestamps Map 무한 증가 → 장기 운영 시 OOM | 1시간 주기 forEach 클린업 추가 |
| L-2 | ✅ 완료 | `conversations/[id]/page.tsx` L237 | 스트리밍 중 탭 전환 후 복귀 시 임시 메시지 + API 메시지 중복 표시 | 탭 복귀 시 temp 메시지 제거 후 loadConv() |
| L-3 | ✅ 완료 | `conversations/[id]/page.tsx` L400 | 600ms 디바운스 중 탭 닫으면 핵심 메모리 수정사항 유실 | beforeunload 시 즉시 flush (keepalive fetch) |

---

## 🟠 High — 성능

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| P-1 | ✅ 완료 | `conversations/new/page.tsx` L37 | /api/tags · /api/stat-tags · /api/characters 순차 fetch → 로딩 3배 느림 | Promise.all() 병렬화 |
| P-2 | ✅ 완료 | `app/api/conversations/route.ts` L9 | 목록 조회 시 캐릭터 전체 컬럼 include → 100개 대화면 200개 캐릭터 풀로딩 | select로 필요 컬럼만 (id, name, avatarUrl) |
| P-3 | ✅ 완료 | `components/ui/CharacterForm.tsx` L50 | 폼 마운트마다 /api/names · /api/persona-tags 재fetch | 모듈 레벨 캐시로 세션 내 1회만 fetch |
| P-4 | ✅ 완료 | 어드민 전체 API 라우트 | findMany() limit 없음 → 대용량 시 DB 프리즈 | take 한도 추가, images에 페이지네이션 추가 |

---

## 🟠 High — 에러 처리

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| E-1 | ✅ 완료 | `conversations/[id]/page.tsx` L165 | lorebook·memory fetch 실패를 .catch(()=>{}) 로 묻음 | 로드 실패 상태 별도 표시 |
| E-2 | ✅ 완료 | `conversations/[id]/page.tsx` L713 | 에러 닫기 후 재전송 시 같은 에러 반복, 재시도 버튼 없음 | "재시도" 버튼 추가 |
| E-3 | ✅ 완료 | `chat/route.ts` SSE | 스트리밍 에러에 status code 없어 클라이언트가 재시도 여부 판단 불가 | { error, retryable: boolean } 포함 |

---

## 🟡 Medium — UI/UX

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| U-1 | ✅ 완료 | `conversations/[id]/page.tsx` | 스트리밍 시작 전 3초+ 대기 시 점(…)만 표시 | 3초 이후 "N초째 생성 중..." 텍스트 표시 |
| U-2 | ✅ 완료 | `conversations/[id]/page.tsx` | fixed popup이 소형 모바일(<375px)에서 잘림 | min/maxWidth: min(Xpx, 90vw) |
| U-3 | ✅ 완료 | `chatlist/page.tsx` | 에러 상태와 빈 상태 메시지 동일 | 에러 전용 안내 화면 분리 |
| U-4 | ✅ 완료 | `conversations/[id]/page.tsx` | 스트리밍 중 30분 이상 응답 없어도 연결 유지 | 30초 무데이터 시 자동 중단 + 재시도 버튼 |

---

## 🟡 Medium — 코드 품질

| # | 상태 | 파일 | 문제 | 개선 방향 |
|---|---|---|---|---|
| C-1 | ✅ 완료 | `chat/route.ts` L114 | conv.currentAI as 'gemini' 하드코딩 | AIProvider 타입 사용 |
| C-2 | ✅ 완료 | 주요 API 라우트 | req.json() 스키마 검증 부족 | conversations POST에 길이·타입 검증 추가 |
| C-3 | ✅ 해당없음 | API 전체 | 에러 응답 형식 라우트마다 다름 | 전체 스캔 결과 이미 `{ error }` 형식으로 통일됨 |
| C-4 | ✅ 완료 | `chat/route.ts` tikiTaka | 캐릭터 응답 순서가 DB 지연으로 뒤바뀔 수 있음 | 메시지 createdAt에 100ms 오프셋 적용 |
