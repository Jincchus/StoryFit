# 로판AI(rofan.ai) 국내 센터 추가 — 설계 노트

> 작성일: 2026-06-19
> 상태: 승인 완료 → 구현
> 선행: [Chub 외국 센터](2026-06-19-chub-foreign-center-import.md) 패턴 재사용(번역 모듈만 제외)

## 결론
- **국내(한국어) 센터** → 번역 불필요. Melting/Zeta/Tikita처럼 capture만 추가.
- 데이터는 **비로그인 SSR `__NEXT_DATA__` JSON**에 전부 공개 → 헤드리스·쿠키·API키 불필요, 결정적.

## 데이터 출처 (확정)
- 캐릭터 URL: `https://rofan.ai/character/{uuid}` (og:url·canonical 확인, 홈 카드가 전부 이 링크)
- 페이지 HTML의 `<script id="__NEXT_DATA__">` → `props.pageProps`:
  - `oriBotDetail` — 캐릭터 본체
  - `botTags[]` — `{ tag_name }` (이미 한국어)
  - `botAssets[]` — 추가 이미지(감정별)
- 도입부에 `Guest`/`{{user}}`/`{{char}}` 매크로 없음 → 치환 불필요
- 예시대화 필드 없음 → exampleDialogues `''`

## 필드 매핑 (oriBotDetail → AssembledCharacter/AssembledResult)

| rofan | → 우리 필드 | 비고 |
|---|---|---|
| `char` | name | |
| `gender` (male/female/other) | 남성/여성/'' | |
| `char_persona` | additionalInfo (본문) | |
| `worldview` | additionalInfo append `[세계관]` | |
| `creator_message` | additionalInfo append `[제작자 메모]` | |
| `first_message` | openingMessage (단일) | |
| `summary` | scenarioDescription (컬렉션 태그라인) | |
| `botTags[].tag_name` | tags | 한국어 그대로 |
| `char_image` | avatarUrl + coverImageUrl | 외부 URL(v1 정책 부합) |
| `nsfw` | safetyLevel = nsfw ? 'relaxed' : 'standard' | |
| (없음) | exampleDialogues `''` | |

## 구현
- `lib/import/rofan.ts` — `captureRofan(url)`: uuid 추출 → 페이지 fetch → `__NEXT_DATA__` 파싱 → assembledResult 직접 조립 → runImport
- route.ts: `matchesHost(url, 'rofan.ai')` 분기
- collections·characters API: `isRofan` 소스 필터(`sourceUrl contains 'rofan.ai'`)
- **전용 센터 페이지 풀세트** (Chub과 동급): `(rofan)` 라우트 그룹 + `.rofan-*` CSS(로즈 `#e0529c`) + 홈/탐색/채팅목록/가이드/생성·편집/센터태그/Dock 통합
- 내부 키 `rofan` / 라우트 `/rofan` / 표시명 "로판AI" / 배지 `ROFAN`
- 단일 캐릭터 구조 → Melting/Chub 상세 미러
- 테스트: `lib/import/rofan.test.ts` — parseRofanUrl + assembleFromNextData 순수 함수
