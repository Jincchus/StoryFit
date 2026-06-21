# babechat / loveydovey 센터 추가 — 조사·설계 노트

> 작성일: 2026-06-19
> 선행: [Chub](2026-06-19-chub-foreign-center-import.md), [rofanai](2026-06-19-rofan-center-import.md)

두 센터 모두 캐릭터 데이터가 **로그인 인증 뒤**에 있어, 공개 URL fetch만 하는
rofan/chub과 다른 부류임을 조사로 확인. (분석은 운영자 본인 계정 토큰으로 수행)

## loveydovey (loveydovey.ai) — 메타데이터 한정
- 데이터: Firebase(reelso-prod) Firestore. `Characters/{id}` 문서는 **공개 읽기 허용**.
  → 비로그인 REST(`firestore.googleapis.com/.../Characters/{id}?key={공개apiKey}`)로 메타 획득.
- ⚠️ persona·도입부(first_message)는 **인증해도 추출 불가**: 서브컬렉션 listCollectionIds 403,
  콘텐츠 API 경로 없음(Firebase ID 토큰으로도 동일). 서버 보호로 판단.
- 다국어: `translatedInstructionInfos.{lang}`에 name/description/job/age. ko 우선 사용.
- 매핑: name·description(태그라인→scenarioDescription)·job·age·primaryGenre(태그)·chatbotImageUrl·
  ageRestriction(→safetyLevel). 도입부·설정은 빈값 → 사용자가 edit에서 작성.
- 구현: `lib/import/loveydovey.ts` (토큰 불필요). 전용 센터 풀세트(coral 테마).

## babechat (babechat.ai) — 인증 API, 풀 데이터
- 데이터: 페이지는 Clerk 게이트(비로그인 404). 실제 API는 **`api.babechatapi.com/{locale}/api`**.
  - 캐릭터 상세: `GET /ko/api/characters/{id}` + `Authorization: Bearer {access}`
  - 토큰 갱신: `POST /ko/api/auth/token/refresh?refresh_token={refresh}`
- 토큰: 운영자 로그인 토큰을 `globalConfig`(babechat_access_token / babechat_refresh_token)에 저장.
  관리자 "가져오기 인증" 화면에서 입력. 액세스 토큰(~3일) 만료 시 refresh로 자동 갱신.
- 매핑: name·description(→scenarioDescription)·targetGender(→gender)·tags·isAdult(→safetyLevel)·
  characterDetails(details/jobs/height/interests/likes/dislikes/location → additionalInfo)·
  startingScenarios[](initialAction 상황 + initialMessage → 도입부, `{{user}}` 매크로 보존).
- 구현: `lib/import/babechat.ts`. 전용 센터 풀세트(indigo 테마).

## 법적/주의
- 운영자 본인 계정의 인증 접근(개인 사용). 자동 갱신은 본인 세션 유지(공식 refresh 엔드포인트).
- ToS의 자동화 조항 위반 소지 + 가져온 캐릭터는 원작자 저작물 → 재배포·공개 지양. 기존 Melting 쿠키 방식과 동일 선상.
