# 레이아웃 재설계 09 — 설정·관리자 다크 점검 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 설정·관리자 페이지는 레이아웃을 유지한 채 다크 모드에서 깨지는 부분만 점검·보정한다.

**Architecture:** Plan 01에서 다크가 `:root` 기본이 되었으므로 대부분 자동 적용된다. 라이트 전제(흰 배경/검정 텍스트 하드코딩, `rgba(0,0,0,0.05)` 등) 잔재만 토큰으로 교체한다. 설정 테마 탭은 Plan 01에서 이미 제거됨.

**Tech Stack:** Next.js 14, CSS.

**선행:** Plan 01·02.

**검증:** 수동 확인(설정 5탭 + 관리자 10개 페이지 가독성), `npx next build`.

---

## File Structure
- 수정(필요 시): `apps/web/app/(main)/settings/_components/*.tsx`, `apps/web/app/(main)/admin/**/*.tsx`, `apps/web/app/globals.css`

---

### Task 1: 하드코딩된 라이트 색상 스캔

- [ ] **Step 1: 라이트 전제 색상 검색**

Run: `cd apps/web && grep -rn "rgba(0,0,0,0\.\|#fff\b\|#ffffff\|#000\b\|#000000\|background:\s*white\|color:\s*black" "app/(main)/settings" "app/(main)/admin"`
결과 목록을 만들어 각 항목이 다크에서 가독성 문제를 일으키는지 판단.

- [ ] **Step 2: 토큰으로 교체**

발견된 하드코딩을 의미에 맞는 토큰으로 교체:
- 배경 흰색 → `var(--paper)` 또는 `var(--pane)`
- 검정 텍스트 → `var(--ink)`
- `rgba(0,0,0,0.05)` 류 옅은 배경 → `var(--pane)` 또는 `rgba(255,255,255,0.04)`
- 보더 → `var(--chrome-border)` / `var(--hairline)`

(ProfileTab의 `rgba(0,0,0,0.05)` 관리자 규칙 박스, ExportTab/StatsTab의 `rgba(0,0,0,0.04)` 안내 박스 등)

- [ ] **Step 3: 빌드 검증**

Run: `cd apps/web && npx next build`
Expected: 성공.

---

### Task 2: 수동 가독성 점검

- [ ] **Step 1: 설정 5탭 확인**

`npm run dev` → 🙂 메뉴 → 설정: 프로필·프롬프트 / 파라미터 / 보안 / 통계 / 내보내기 각 탭이 다크에서 텍스트·입력·슬라이더·카드 가독성 정상. (테마 탭 없음 확인)

- [ ] **Step 2: 관리자 10개 페이지 확인**

🙂 메뉴 → 관리자: 대시보드/랜덤 이름/태그/전역 설정/쿠키 가져오기/유저/이미지/AI 비용/활동 로그/오류 로그 — 표·폼·차트가 다크에서 읽히는지 확인. 깨진 항목만 Task 1 방식으로 보정.

- [ ] **Step 3: 커밋**

```bash
git add -A
git commit -m "Style: 설정·관리자 다크 모드 가독성 보정(레이아웃 유지)"
```

---

## Self-Review
- 스펙 "설정·관리자 레이아웃 유지 + 다크 CSS만" → Task 1~2.
- 유실 없음(레이아웃·기능 불변, 색상만 보정).

---

# 전체 플랜 인덱스 (레이아웃 재설계)
1. 01 다크 단일화 + 테마/배경 제거 ← 선행
2. 02 전역 셸(헤더·계정 메뉴·독)
3. 03 홈
4. 04 채팅목록
5. 05 탐색
6. 06 서재
7. 07 채팅방 + 설정 패널
8. 08 챗봇
9. 09 설정·관리자 다크 점검

권장 실행 순서: 01 → 02 → (03~08 임의 순서, 04가 convDisplay 헬퍼 생성하므로 03보다 먼저 또는 03 인라인 사용) → 09.
의존성 메모: 03 홈과 06 서재는 04가 만드는 `lib/convDisplay.ts`를 사용하면 중복이 줄어든다. 04를 03보다 먼저 실행하거나, 03의 인라인 헬퍼를 04 완료 후 import로 교체.
