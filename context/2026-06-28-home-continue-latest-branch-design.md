# 홈 이어하기 — 분기 최신 기준 + 분기 직접 진입 설계

작성일: 2026-06-28
대상: 신규 `apps/web/app/api/conversations/recent/route.ts`, 수정 `apps/web/app/(main)/page.tsx`

## 배경 / 문제

- 홈 화면 "이어하기"(최근 3개)는 `GET /api/conversations`를 쓰는데, 이 라우트는 `rootConversationId: null`(root 대화)만 반환한다.
- 미리보기 메시지는 `messages: { orderBy: createdAt desc, take 1 }`로 **isSelected 필터 없이** 전역 최신 메시지를 가져온다.
- 결과: 분기(별도 Conversation, `rootConversationId`로 연결)에서 마지막으로 대화했어도 홈엔 **root(오래된) 대화가 뜨고, 클릭하면 root로 진입**. 미리보기도 활성 분기가 아닐 수 있음.
- 분기는 각자 고유 id·updatedAt을 가지며 `/conversations/{분기id}`로 직접 진입 가능(분기 스위처가 이미 그렇게 동작).

## 목표 (사용자 확정, 홈 이어하기만)

- 이어하기는 대화별로 **가장 최근 활동한 분기**(root 포함)를 보여준다.
- 카드 클릭 시 **그 분기로 직접 진입**한다.
- 미리보기는 그 분기의 **활성 경로(isSelected) 최신 메시지**.
- 채팅목록·공용 `/api/conversations`·기타 화면은 변경하지 않는다.

## 설계 (홈 전용 엔드포인트 신설)

### 신규 `GET /api/conversations/recent?limit=N`
1. 사용자 모든 대화 노드(root + 분기)를 **경량 조회**: `where: { userId, isArchived: false, mode: { not: 'assistant' } }`, `select: { id, rootConversationId, updatedAt }`.
2. **스레드 그룹화**: 스레드키 = `rootConversationId ?? id`. 스레드마다 `updatedAt` 최대인 노드(가장 최근 활동 분기)를 선택.
3. 선택 노드들을 `updatedAt` desc 정렬 → 상위 `limit`개 id 추출(`limit` 기본 3, 상한 20).
4. 그 id들만 **풀 조회**: `include: { characters: { include: { character: { select: { id,name,avatarUrl } } } }, messages: { where: { isSelected: true }, orderBy: { createdAt: 'desc' }, take: 1 }, personaCharacter: { select: { name: true } } }`.
5. 3단계 정렬 순서를 유지해 반환(홈이 기대하는 형태: `id, title, updatedAt, mode, messages[], characters[]`).

성능: 1단계는 경량(스칼라 3개), 풀 조회는 상위 N개만 → N+1 없음.

### 홈 페이지(`app/(main)/page.tsx`)
- `api.get('/api/conversations')` → `api.get('/api/conversations/recent?limit=3')`.
- 기존 클라이언트 정렬(`updatedAt` desc)·`slice(0,3)`는 엔드포인트가 이미 정렬·제한하므로 그대로 두거나 단순화(그대로 둬도 무해).
- 카드 링크 `router.push('/conversations/' + c.id)` — 이제 `c.id`가 최신 분기 id라 그 분기로 직접 진입.
- 미리보기 `c.messages[0]?.content` — isSelected 기준이라 활성 분기 최신 줄.

## 검증

- `npx tsc --noEmit`.
- 수동: 분기를 만들어 분기에서 대화 → 홈 이어하기에 그 분기 미리보기가 뜨고, 클릭 시 그 분기(루트 아님)로 진입하는지. 분기 없는 대화는 기존과 동일하게 동작.

## 영향 / 리스크

- 신규 라우트만 추가, 공용 API·chatlist 불변(회귀 위험 낮음).
- 홈 fetch URL 1줄 교체.

## 범위 외 (YAGNI)

- 채팅목록·서재의 분기 기준 변경, 분기 목록 별도 표시.
