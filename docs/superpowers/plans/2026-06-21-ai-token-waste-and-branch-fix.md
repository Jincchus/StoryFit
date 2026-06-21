# AI 토큰 소모 최적화 및 분기(Branch) parentId 복구 구현 계획

## 1. 개요 및 문제 분석

현재 대화 진행 시 입력(in) 토큰이 수십만 개씩 비정상적으로 높게 소모되는 현상이 발견되었습니다. 조사 결과 두 가지 요인이 겹쳐 발생한 것으로 파악되었습니다.

### #1 추천 답변 칩(Suggestions)의 추가 API 호출로 인한 중복 전송
- 기존 6/15 커밋(`68203a6`)에서 4지선다 선택지를 챗 본문 하단에 포함시키는 대신 별도의 `/suggestions` API를 매 턴마다 호출하게 하여 토큰이 중복으로 발생했습니다.
- 따라서, 1회 Gemini API 호출로 본문과 4지선다를 모두 받아오던 이전 방식으로 롤백합니다.

### #2 대화방 분기(Branch) 시 과거 메시지 parentId 유실 버그 (핵심 원인)
- 대화를 중간에 분기(Branch)할 때 복사 대상 메시지들을 새 대화방에 인서트하는 과정에서, 모든 복사 메시지의 `parentId`가 `null`로 세팅되어 저장되는 현상이 있었습니다 (`branch/route.ts`).
- 시스템 프롬프트 구성 시 `splitRecentAndOpening` 헬퍼 함수는 `parentId`가 `null`인 모든 assistant(AI) 메시지를 "도입부/오프닝 지문"으로 인식하고 `openingScene`에 병합합니다.
- 이에 따라, 분기된 대화방에서 대화를 이어 나가면 **최근 대화 버젯을 초과한 과거 대화의 모든 AI 응답이 오프닝 지문 취급을 받아 systemPrompt로 전송**되는 토큰 폭증을 유발했습니다.

---

## 2. 해결 방식 설계

### #1 AI 4지선다 생성 및 파싱 복구
- **프롬프트 복구**: `lib/systemPrompt.ts` 내의 `buildStoryBaseRules` 및 `buildMultiStorySystemPrompt`에 `---` 구분선 및 4지선다 지시문/예시를 복원합니다.
- **API 롤백**: `chat/route.ts` 및 `regenerate/route.ts`의 `allowChoices` 플래그를 모드에 따라 다시 활성화하고 `revisionOptions` 규칙을 복구합니다.
- **프론트엔드 UI 롤백**: `MessageList.tsx`에서 칩(Suggestions)을 제거하고, 본문 파싱 데이터 `storyParsed.choices`를 4지선다 버튼으로 직접 렌더링하는 기존 UI 구조로 되돌립니다. `page.tsx`에서도 매 턴 자동 suggestions를 쏘던 `useEffect`를 제거합니다.

### #2 branch/route.ts 에서 복사 시 메시지 간 parentId 관계 재구축
- nested create 방식은 한 번에 데이터가 인서트되어 생성 시점 이전의 메시지들 간 ID를 알 수 없었기 때문에 `parentId: null`로 복사했었습니다.
- 이를 **순차 루프 및 ID 매핑 맵**을 사용하여 개별 `prisma.message.create`로 변경합니다.
- 복사 시 `oldToNewIdMap: Map<string, string>`을 통해 부모 메시지의 새로운 ID를 맵에서 가져와 `parentId`에 매핑합니다.
- 결과적으로 분기된 대화방에서도 메시지 위계 구조가 정상 보존되며, 진짜 오프닝 지문을 제외한 과거 메시지는 `openingScene`에 취합되지 않아 토큰 폭증이 발생하지 않게 됩니다.

---

## 3. 구현 세부 계획

### 변경 대상 파일
1. **`lib/systemPrompt.ts`**: 4지선다 아웃풋 예시 및 지시 규칙 복원
2. **`app/api/conversations/[id]/chat/route.ts`**: `allowChoices = true` 복구 및 `revisionOptions` 원복
3. **`app/api/conversations/[id]/regenerate/route.ts`**: `allowChoices = true` 복구 및 `revisionOptions` 원복
4. **`app/api/conversations/[id]/branch/route.ts`**: 순차 메시지 복사 및 `parentId` 매핑 저장 보존 로직 추가
5. **`app/(main)/conversations/[id]/_components/MessageList.tsx`**: 4지선다 버튼 리스트 UI 복구 및 suggestions 제거
6. **`app/(main)/conversations/[id]/page.tsx`**: suggestions useEffect 훅 및 관련 Props 제거

---

## 4. 검증 계획

- **정적 분석**: `npx tsc --noEmit`을 통한 컴포넌트 Props 검사
- **빌드 테스트**: `npm run build` 성공 여부
- **동작 검증**:
  - 스토리 모드 진행 시 AI 본문 하단에 4지선다 버튼이 잘 나타나는지 확인.
  - 대화 분기 생성 직후 systemPrompt 내용과 `inputTokens` 크기가 평소처럼(6~8천) 유지되는지 관찰.
