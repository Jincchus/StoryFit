# 팅글(tingle.chat) import 개선 & 토큰 자동 갱신 (2026-06-22)

## 1. 팅글 API 구조 확인

팅글은 WHIF와 달리 캐릭터-서사-테마 간 실질적 1:1 연결이 없음.
- `GET /universes?personaId={id}`, `GET /scenes?personaId={id}` → 어떤 캐릭터 ID를 넣어도 전체 목록 25개 반환
- 캐릭터/서사/테마는 독립적으로 등록하고 상세 페이지에서 사용자가 직접 조합

팅글 URL 형식:
- 캐릭터: `https://tingle.chat/chat/characters/{id}`
- 서사: `https://tingle.chat/chat/universes/{id}`
- 테마: `https://tingle.chat/chat/scenes/{id}`

팅글 API:
- 캐릭터: `GET https://api.tingle.chat/personas/{id}`
- 서사: `GET https://api.tingle.chat/universes/{id}`
- 테마: `GET https://api.tingle.chat/scenes/{id}`
- 인증: `Authorization: Bearer {Firebase ID Token}`

---

## 2. import 미리보기 2단계 플로우

### 변경 전
URL 입력 → 즉시 저장

### 변경 후
URL 입력 → **미리보기 모달** → 필드 편집 후 저장

### 신규 엔드포인트
`POST /api/characters/import/preview`
- 팅글 URL → 저장 없이 raw fields 반환
- 파일: `app/api/characters/import/preview/route.ts`

### 미리보기 모달 기능 (`app/(tingle)/tingle/page.tsx`)
- 각 필드에 **X 버튼** → 제거(removed=true), 다시 클릭 시 복원
- 각 필드에 **숫자 입력** → 표시 순서 지정 (order)
- 도입부(openings)도 X 버튼으로 제거 가능
- "완료" 클릭 시 `POST /api/characters/import` 에 `{ url, previewData }` 전달

### import/route.ts 변경
- `buildCapturedFromPreview(previewData)`: 편집된 TingleRawData → Captured 변환
  - `fields.filter(f => !f.removed).sort((a,b) => a.order - b.order)` 로 순서/제거 반영
  - `openings.filter(o => !o.removed)` 로 제거된 도입부 제외
  - universe 타입이면 `relationships` 필드 → `exampleDialogues`로 저장

---

## 3. linkedItems 제거

초기에 WHIF 패턴 참고해 캐릭터 등록 시 연결 서사/테마 자동 등록 시도했으나,
팅글 API가 연결 목록이 아닌 전체 목록을 반환함을 확인 후 전면 제거.

제거된 항목:
- `TingleLinkedItem` 인터페이스 (`lib/import/types.ts`)
- `fetchTingleLinked()` 함수 (`lib/import/capture.ts`)
- `TingleRawData.linkedItems?` 필드
- `import/route.ts`의 linkedItems 순차 import 블록

---

## 4. SelectList "URL로 추가" 기능

캐릭터/서사/테마 상세 페이지에서 연계 항목이 없거나 추가하고 싶을 때,
각 SelectList 하단 `+ URL로 추가` 버튼으로 팅글 URL 직접 입력해 등록.

적용 파일:
- `app/(tingle)/tingle/characters/[id]/page.tsx`
- `app/(tingle)/tingle/universes/[id]/page.tsx`
- `app/(tingle)/tingle/scenes/[id]/page.tsx`

각 페이지의 `handleAddUrl`:
```typescript
const handleAddUrl = async (url: string) => {
  await api.post('/api/characters/import', { url })
  const all = await api.get('/api/collections?isTingle=true')
  setAllTingle(all)
}
```

SelectList 컴포넌트에 `onAddUrl?: (url: string) => Promise<void>` prop 추가.
항목이 있어도 항상 표시됨 (언제든 추가 가능).

---

## 5. 팅글 Firebase 토큰 자동 갱신

### 배경
팅글은 Firebase Authentication 기반. ID 토큰(JWT)은 발급 후 1시간 만료.
기존에는 만료마다 관리자 페이지에서 수동 갱신 필요.

### 구현
Firebase refresh token + API key를 GlobalConfig에 저장 후 만료 시 자동 갱신.

**새 GlobalConfig 키:**
- `tingle_refresh_token`: Firebase refresh token (로그아웃 전까지 만료 없음)
- `tingle_firebase_api_key`: Firebase Web API key (`AIzaSy...`)

**갱신 엔드포인트:**
```
POST https://securetoken.googleapis.com/v1/token?key={API_KEY}
Content-Type: application/x-www-form-urlencoded
body: grant_type=refresh_token&refresh_token={REFRESH_TOKEN}
응답: { id_token: "새 ID 토큰", refresh_token: "새 refresh token", ... }
```

**`capture.ts` 변경 (`fetchTingleData`):**
1. 저장된 ID 토큰 만료 여부 확인 (`isTingleTokenExpired`)
2. 만료됐으면 `refreshTingleToken()` 호출 → 새 토큰 발급 + DB 저장
3. 요청 후 401/403이면 한 번 더 refresh 시도 (클럭 오차 대응)

**값 취득 방법:**
- Firebase API 키: DevTools → Network → `securetoken.googleapis.com/v1/token?key=AIzaSy...` URL 확인
- Refresh token: 같은 요청의 Payload 탭 → `refresh_token` 값

**현재 저장된 값:** DB에 직접 저장 완료 (2026-06-22)

### 어드민 UI
`app/(main)/admin/import-cookies/page.tsx`에서 팅글 섹션은 기본 **접힘** 상태.
"자동 갱신 설정됨 — 수동 변경 필요 시 펼치기" 라벨로 표시.
▼ 클릭 시 3개 필드(ID 토큰, refresh 토큰, API 키) 펼쳐짐.

---

## 6. 팅글 화면 구조

### `/tingle` 목록 페이지
- 캐릭터/서사/테마 탭 구분
- 상단 "🔍 미리보기" 버튼 → URL 입력 → 미리보기 모달 → 저장

### 캐릭터 상세 (`/tingle/characters/[id]`)
- 캐릭터 설정(additionalInfo), 도입부 표시
- 서사 선택 / 테마 선택 (SelectList, URL로 추가 가능)
- "대화 시작" → 선택한 서사/테마 설명을 scenarioDescription에 합산

### 서사 상세 (`/tingle/universes/[id]`)
- 서사 설정(additionalInfo), 관계 설정(exampleDialogues) 표시
- 캐릭터 선택 (도입부 미리보기 포함) / 테마 선택
- "대화 시작" → `{캐릭터} × {서사}` 제목으로 대화방 생성

### 테마 상세 (`/tingle/scenes/[id]`)
- 테마 설명(additionalInfo) 표시
- 캐릭터 선택 (도입부 미리보기 포함) / 서사 선택
- "대화 시작" → `{캐릭터} × {테마}` 제목으로 대화방 생성

---

## 7. 관련 파일 목록

| 파일 | 변경 내용 |
|------|-----------|
| `lib/import/types.ts` | TingleField, TingleOpening, TingleRawData 정의. TingleLinkedItem 제거 |
| `lib/import/capture.ts` | captureTingleRaw(), fetchTingleData(), refreshTingleToken(), isTingleTokenExpired() |
| `app/api/characters/import/preview/route.ts` | 신규 — 미리보기 엔드포인트 |
| `app/api/characters/import/route.ts` | buildCapturedFromPreview(), previewData 지원 추가 |
| `app/api/admin/import-cookies/route.ts` | tingle_refresh_token, tingle_firebase_api_key 키 추가 |
| `app/(tingle)/tingle/page.tsx` | ImportPreviewModal, handlePreview, handleConfirm |
| `app/(tingle)/tingle/characters/[id]/page.tsx` | SelectList onAddUrl 지원 |
| `app/(tingle)/tingle/universes/[id]/page.tsx` | SelectList onAddUrl 지원 |
| `app/(tingle)/tingle/scenes/[id]/page.tsx` | SelectList onAddUrl 지원 |
| `app/(main)/admin/import-cookies/page.tsx` | 팅글 섹션 접기, 새 필드 추가 |
