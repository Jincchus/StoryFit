# 핵심메모리 승격 해제(되돌리기) — 설계

작성일: 2026-06-17

## 배경 / 문제

채팅방 설정탭의 **장기 메모리** 패널에서 자동 요약된 메모리(`Memory`)를 선택해
**핵심메모리**(`Conversation.coreMemory`)로 승격할 수 있다. 승격 흐름:

1. 10턴마다 자동 요약 → `Memory` 행 생성 (`promoted: false`)
2. 체크박스로 선택 → "↑ 핵심 메모리로 올리기"
3. `memories/promote` POST: 선택 요약을 `condenseForCoreMemory`로 압축·병합 → `coreMemory` 텍스트에 append → 원본 `Memory`를 `promoted: true`로 마킹
4. 승격된 메모리는 UI에서 체크박스가 사라지고("↑ 핵심"), 접힘/반투명 처리되어 **다시 선택 불가**

`promoted`가 되돌릴 수 없는 단방향 플래그라서, AI가 핵심메모리를 잘 인식하지 못해
핵심메모리를 정리하고 원본 요약을 다시 올리려 해도 잠겨 있어 재승격이 불가능하다.

## 목표

이미 승격된 메모리의 잠금을 풀어(승격 해제) 나중에 다시 핵심메모리로 올릴 수 있게 한다.

## 설계

### 핵심 결정

- 승격 해제는 **원본 메모리(`Memory.promoted`)의 잠금만 푸는** 단방향 정리 도구다.
- 핵심메모리 텍스트(`coreMemory`)는 **건드리지 않는다.** 승격 시 여러 요약이 AI로 압축·병합되어
  원본 메모리와 텍스트 조각의 연결이 끊겨 있으므로, 텍스트에서 특정 메모리만 제거하는 것은
  불확실하고 비용이 든다. 핵심메모리 텍스트 관리(직접 편집 / AI 재압축)는 기존 UI를 그대로 활용한다.
- 스키마 변경 없음 (`Memory.promoted` Boolean을 되돌릴 수 있게만 만든다).

### 사용자 흐름

1. AI가 핵심메모리를 잘 인식 못 함 → 핵심메모리 텍스트를 직접 정리하거나 AI 재압축 버튼 사용
2. 다시 올리고 싶은 원본 요약의 "↑ 핵심" 카드에서 **↩ 해제** 클릭 → 체크박스 부활
3. 정리된 상태에서 원하는 항목 재선택 → 기존 "↑ 핵심 메모리로 올리기"로 재승격
   - ※ 핵심메모리 텍스트를 비우지 않고 재승격하면 내용이 중복 append 된다. 이는 의도된 동작이며,
     사용자가 텍스트를 정리하거나 AI 재압축으로 중복을 제거할 수 있다.

### 변경 범위

| 레이어 | 파일 | 변경 |
|--------|------|------|
| API | `app/api/conversations/[id]/memories/promote/route.ts` | `DELETE` 메서드 추가 — body의 `memoryIds`(string[], 1~20)를 받아 해당 대화의 메모리를 `promoted: false`로 되돌림. `coreMemory`는 미변경. 소유권/검증 로직은 기존 POST와 동일 패턴 |
| Hook | `app/(main)/conversations/[id]/_hooks/useMemoryPanel.ts` | `handleUnpromoteMemory(id)` 추가 — `DELETE /memories/promote` 호출 성공 시 로컬 state에서 해당 메모리 `promoted: false`로 갱신. 실패 시 토스트 |
| UI | `app/(main)/conversations/[id]/_components/SidePanel.tsx` | "↑ 핵심" 카드 헤더(✕ 옆)에 **↩ 해제** 버튼 추가. 클릭 시 `e.stopPropagation()`으로 접힘 토글과 분리하고 `handleUnpromoteMemory(mem.id)` 호출 |
| 가이드 | `app/(main)/guide/page.tsx` | `FEATURE_SECTIONS`에 핵심메모리 승격 해제 항목 반영 (CLAUDE.md 사용자 기능 가이드 동기화 규칙) |

### API 상세

`DELETE /api/conversations/[id]/memories/promote`

- 인증: `authenticate(req)` — 미인증 401
- 대화 소유권 확인 — 없거나 타인 소유 시 404
- body 파싱 실패 시 400
- `memoryIds`: 배열, 1~20개, 모두 string이어야 함 — 아니면 400
- `prisma.memory.updateMany({ where: { id: { in: memoryIds }, conversationId }, data: { promoted: false } })`
- 응답: `{ unpromotedIds: string[] }`

### 에러 처리

- API: 위 검증 단계별 상태코드 반환. 트랜잭션 불필요(단일 updateMany).
- Hook: API 실패 시 로컬 state 미변경 + "핵심 메모리 해제에 실패했습니다" 토스트.

### 테스트 / 검증

- 승격된 메모리에서 ↩ 해제 → 체크박스 부활 확인
- 해제 후 재선택 → 재승격 가능 확인
- 핵심메모리 텍스트는 해제 전후 동일함 확인
- 타인 대화 ID로 DELETE 호출 시 404 확인
