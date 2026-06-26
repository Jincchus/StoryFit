# 새 센터 추가 매뉴얼 / 체크리스트

외부 가져오기 센터(WHIF·ZETA·MELTING·TIKITA·CHUB·rofanai·loveydovey·babechat·tingle …)를
새로 추가할 때 손대야 하는 모든 지점을 정리한다. **단일 소스인 `apps/web/lib/centers.ts`를
먼저 고치면 일부는 자동 반영**되고, 나머지는 센터 고유라 수동으로 추가해야 한다.

> 작업 시작 전: 기존 센터 1개(예: `rofan` — 단독 등록형 / `tingle` — 다중 타입형)를
> 레퍼런스로 잡고 그 파일들을 복사·치환하는 방식이 가장 안전하다.

---

## 0. 단일 소스 — 가장 먼저

**`apps/web/lib/centers.ts`** 의 `CENTERS` 배열에 새 항목을 추가한다.

```ts
{
  key: 'newcenter',            // 라우트/식별 키. 디렉터리 app/(newcenter)/newcenter/ 와 일치
  label: 'NewCenter',          // 화면 표시 라벨(브랜드 표기)
  domain: 'newcenter.com',     // 대표 도메인
  path: '/newcenter',
  emoji: '🆕',
  grad: 'linear-gradient(135deg, #aaa, #bbb)',
  desc: '한 줄 설명',
  dbHosts: ['newcenter.com'],  // collections API sourceUrl contains 필터
  importHosts: ['newcenter.com'], // import matchesHost용(참고)
  cssVar: '--nc-', cssClass: 'newcenter-', storagePrefix: 'nc', // 실제 사용 접두사 기록
}
```

---

## 1. 자동 반영 (레지스트리만 고치면 됨 — 추가 작업 X)

아래는 `CENTERS`를 순회하므로 0번만 하면 따라온다:

| 파일 | 역할 |
|------|------|
| `app/api/collections/route.ts` | `isXxx=true` → sourceUrl 호스트 필터 |
| `app/api/characters/route.ts` | 동일(캐릭터 단위) |
| `components/shell/Dock.tsx` | 탐색 탭 active 판정 (`CENTER_PATHS`) |
| `app/(main)/explore/page.tsx` | 센터 카드 그리드 |

---

## 2. 수동 작업 (센터 고유 — 직접 추가)

### A. 라우트 / 페이지
- `app/(newcenter)/layout.tsx` — 기존 레이아웃 복사, 이름·`newcenter-root` 클래스만 변경
- `app/(newcenter)/newcenter/page.tsx` — 리스트 페이지
- `app/(newcenter)/newcenter/<상세세그먼트>/[id]/page.tsx` — 상세 페이지
  - 상세 세그먼트는 센터마다 다름: whif `characters`/`universes`, zeta `plots`,
    tikita `story`, tingle `characters`/`universes`/`scenes`, 나머지 `characters`

### B. import 파이프라인
- `lib/import/newcenter.ts` — `captureNewcenter(url)` 작성 (`Captured` 반환)
- `app/api/characters/import/route.ts` — `matchesHost(url, ...)` 분기 + import 추가
- 미리보기 흐름이 필요하면 `app/api/characters/import/preview/route.ts`

### C. CSS
- `app/globals.css` — `--<prefix>-*` 변수 세트 + `.<prefix>-*` 클래스 세트 추가
  - ⚠ 접두사 3축(`cssVar`/`cssClass`/`storagePrefix`)이 기존 센터마다 정렬돼 있지 않다.
    새 센터는 가급적 **세 축을 같은 키로 통일**해 추가할 것(부록 표 참고).

### D. 가져오기 인증 (쿠키·토큰이 필요한 센터만)
- `app/api/admin/import-cookies/route.ts` — `KEYS` 배열에 세션 키 추가
- `app/(main)/admin/import-cookies/page.tsx` — 입력 UI
- `lib/import/capture.ts` — `getGlobalConfigValue`의 key union 타입에 추가
- (좋아요/즐겨찾기 일괄 가져오기 지원 시) `app/api/<center>/liked-scan/route.ts`
  + 리스트 페이지에 패널 UI (멜팅/팅글 참고)

### E. 탐색 / 목록 부가 표시
- `app/(main)/explore/all/page.tsx` — 전체보기 센터 배열(`match`/`color`/`detail` 라우트)
- `app/(main)/chatlist/page.tsx` — `sourceUrl → KEY` 감지 + 필터 칩 목록
- `app/(main)/page.tsx` — 홈 "외부 가져오기" 카드 (큐레이션된 설명 문구라 수동 유지)
- `app/(main)/guide/page.tsx` — 사용자 기능 가이드(`FEATURE_SECTIONS`)
  - CLAUDE.md 규칙: 사용자 체감 기능 추가/변경 시 가이드 동기화 필수

### F. 캐릭터 직접 생성/편집 연동 (해당 센터가 isXxx 파라미터를 쓰는 경우)
- `app/(main)/characters/new/page.tsx`
- `app/(main)/characters/[id]/edit/page.tsx`

### G. 관리자 태그 설명
- `app/(main)/admin/center-tags/page.tsx` — 안내 문구의 센터 나열

---

## 3. 작업 후 검증 체크리스트

- [ ] **플레이스홀더 치환**: 상세 페이지의 모든 렌더 필드에
      `replaceDisplayPlaceholders(text, userName, charNames)` 적용 (CLAUDE.md 규칙).
      grep으로 누락 점검:
      `grep -n "{[^}]*}" page.tsx | grep -v "replaceDisplay\|NovelText\|style\|className\|key\|src\|alt\|href\|onClick\|onChange\|disabled\|type\|placeholder"`
- [ ] **useRefetchOnForeground**: 상세 페이지에 적용 (포그라운드 복귀 시 새로고침)
- [ ] **도입부 수정(✏ 편집)**: 다른 센터와 동일한 인라인 편집 UX 제공
- [ ] `npx tsc --noEmit` 통과
- [ ] 리스트→상세→대화 시작 플로우 수동 확인
- [ ] Dock 탐색 탭이 새 센터 경로에서 하이라이트되는지
- [ ] `docker compose up --build` 빌드 성공

---

## 부록: 현재 접두사 불일치 표 (참고)

한 센터 안에서도 3축이 안 맞는 곳이 있다(예: tingle = `tg`/`--tg-`/`tingle-`).
새 센터는 통일해서 추가하길 권장. 값은 `lib/centers.ts`에 기록돼 있다.

| 센터 | storagePrefix | cssVar | cssClass |
|------|------|------|------|
| whif | `whif` | `--w-` | `whif-` |
| zeta | `zeta` | `--z-` | `zeta-` |
| melting | `melting` | `--m-` | `melting-` |
| tikita | `tikita` | `--t-` | `tikita-` |
| chub | `chub` | `--c-` | `chub-` |
| rofan | `rofan` | `--r-` | `rofan-` |
| loveydovey | `lovey` | `--l-` | `lovey-` |
| babechat | `bc` | `--b-` | `bc-` |
| tingle | `tg` | `--tg-` | `tingle-` |
