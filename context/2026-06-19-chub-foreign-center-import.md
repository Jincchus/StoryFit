# Chub.ai 외국 센터 추가 + AI 번역 가져오기 — 설계 노트

> 작성일: 2026-06-19
> 상태: ✅ 구현 완료 (백엔드 + 번역 모듈 + 전용 센터 페이지 풀세트). tsc·lint·테스트(43)·next build 통과.
> ⚠️ 라이브 검증 필요: Chub API 응답 형태(download POST·node JSON)는 알려진 패턴 기반으로 작성 — 실제 호출로 확인 후 보정 요망.

## 목표

현재 프로젝트에 등록된 플랫폼("센터": Melting, Zeta, WHIF, Tikita)에
**외국 센터**를 추가로 붙인다. 첫 대상은 **Chub.ai**.
가져오면서 **AI(Gemini)로 번역**하여 자연스러운 한국어로 등록한다.

## 현재 구조 파악 (확정 사실)

- 모든 센터는 공통 파이프라인 공유: `capture*()` → classify → assemble → DB
- import 라우트는 URL 입력 시 두 경로:
  1. 알려진 센터(zeta/melting/whif/tikita) → 전용 capture 함수
  2. 그 외 → URL 직접 fetch → Tavern Card JSON/PNG 검증
- 국내 센터들은 **Tavern Card 형식 아님** — 각자 자체 포맷
  - Melting: 자체 API(openings 배열·태그), 쿠키 인증 필요
  - Zeta: 자체 구조(plots·intros), `buildZetaCaptured()`
  - WHIF: 헤드리스/API, 자체 포맷
  - Tikita: story_episodes(에피소드/챕터형)
- DB 저장 구조: 가져오기 → `Character` + `CharacterCollection` 레코드 **1회 생성**
  - "센터 페이지"(/melting, /zeta)는 별도 저장소가 아니라 **출처별 필터 뷰**
  - 편집은 `app/(main)/characters/[id]/edit` **한 군데**
    (name, gender, tags, additionalInfo, exampleDialogues, openingMessage)
  - **별도 "센터 내 캐릭터 카드" 사본 없음** — Character 레코드 하나가 단일 출처

## Chub.ai 조사 결과

- 캐릭터 카드 = **chara_card_v2 형식 JSON**
- 가져오는 법:
  - `api.chub.ai/api/characters/{author}/{slug}?full=true` (JSON)
  - 또는 `avatars.charhub.io/.../chara_card_v2.png` 에서 추출
    (`parsePngTavernCard` 이미 보유 → 둘 다 가능)
- 주요 필드: `name`, `description`, `personality`, `scenario`,
  `first_mes`, `mes_example`, `system_prompt`,
  `alternate_greetings[]`(→ 다중 도입부 openingMessages 매핑),
  `tags[]`, `creator_notes`

## 결정 사항 (확정)

| 항목 | 결정 |
|------|------|
| 가져오기 방식 | **URL 붙여넣기** (사용자는 chub URL만 입력, 백엔드가 Chub API 조회 — 기존 센터와 동일 패턴). 별도 검색/브라우즈 UI 안 만듦 |
| 번역 범위 | **전부 번역** — API JSON 중 우리 프로젝트에 넣을 필드는 모두 번역 |
| 편집 시점 | **B안: 저장 후 기존 edit 페이지에서 수정** (별도 미리보기 화면 X). 번역만 capture 단계에 추가 |
| 번역 모델 | **gemini-2.5-pro** 사용. flash 버전도 작성하되 **주석 처리**, `// TODO: flash로 변경 예정` 기재 |
| 캐릭터 이름(name) | **번역 안 함 — 원문 그대로 가져오기.** 음차가 필요하면 사용자가 edit 페이지에서 보고 직접 선택·수정. (식별 쉬움 + 검색 일관성 유지) |
| 태그(tags) | **번역 + 정규화.** 기존 국내 센터의 한글 태그 체계에 매핑 테이블로 맞춤(예: `Sci-Fi`/`SF`→`SF`, `Tsundere`→`츤데레`). 표기 흔들림 방지 |

## 필드 매핑표 (chara_card_v2 → 우리 구조)

목표 구조(`lib/import/types.ts`):
- 캐릭터별 = `AssembledCharacter`: `name, gender, tags[], additionalInfo, openingMessage, openingMessages[], exampleDialogues, avatarUrl, relatedImages[]`
- 컬렉션 레벨 = `AssembledResult`: `scenarioDescription, tags[], title, safetyLevel, coverImageUrl`

| chub 필드 (chara_card_v2) | → 우리 필드 | 번역 | 비고 |
|---|---|---|---|
| `data.name` | `name` | ❌ 원문 | 음차는 사용자가 edit에서 직접 |
| `data.description` | `additionalInfo` (본문) | ✅ | 캐릭터 설정 본체 |
| `data.personality` | `additionalInfo` (이어붙임) | ✅ | `[성격]` 라벨 붙여 append |
| `data.scenario` | `scenarioDescription` (컬렉션) | ✅ | 시나리오는 컬렉션 레벨로 |
| `data.first_mes` | `openingMessage` + `openingMessages[0]` | ✅ | 기본 도입부 |
| `data.alternate_greetings[]` | `openingMessages[1..]` | ✅ | 다중 도입부, `{id,title,content}`로 변환 |
| `data.mes_example` | `exampleDialogues` | ✅ | ⚠️ `<START>`·`{{char}}`·`{{user}}` 마커 **보존**하고 텍스트만 번역 |
| `data.system_prompt` | (기본 무시) | — | 우리 base rules가 대체. 내용 있으면 `additionalInfo`에 `[제작자 지침]`으로 append 검토 |
| `data.creator_notes` | `additionalInfo` (이어붙임) | ✅(선택) | `[제작자 메모]`로 append. 카드 메타성이면 생략 가능 |
| `data.tags[]` | `tags[]` (캐릭터+컬렉션) | ✅ 번역+정규화 | 매핑 테이블 경유, dedup, 최대 15개(타 센터 관행) |
| (gender 필드 없음) | `gender` | — | chara_card_v2엔 성별 필드 없음 → 빈값 기본, tags/description 휴리스틱은 선택 |
| `data.character_book` | `lorebooks[]` | ✅(선택) | 카드 내장 로어북 있으면 변환. 1차 구현은 보류 가능 |
| 아바타 PNG/`avatar` URL | `avatarUrl` | ❌ | `parsePngTavernCard` 추출 가능 |
| nsfw 태그/플래그 | `safetyLevel` | ❌ | 태그 기반 판정 |

### 번역 규칙 (공통)
- **매크로 보존 필수**: `{{char}}`, `{{user}}`, `<START>` 등은 번역하지 말고 그대로 통과.
- **이름(name) 비번역**이지만, 본문 안에 등장하는 캐릭터 이름은 원문 유지(혼동 방지) — 프롬프트에 "고유명사는 원문 유지" 명시.
- **태그 정규화**: `Sci-Fi`/`SF`→`SF`, `Tsundere`→`츤데레` 식 매핑 테이블 + 미정의 태그는 Gemini 번역 폴백.

## captureChub() + 번역 모듈 설계

### 파일 구성
- `lib/import/chub.ts` — `captureChub(url)` (Tikita가 `lib/import/tikita.ts`로 분리된 패턴 따름)
- `lib/import/translate.ts` — 번역 모듈 (capture에서 호출)
- `lib/import/tagMap.ts` — 태그 정규화 매핑 테이블
- `lib/tavernCard.ts` — `TavernCard`에 `tags?`, `alternate_greetings?`, `character_book?` 추가

### 흐름
```
route.ts POST
  → matchesHost(url, 'chub.ai', 'characterhub.org', 'chub.ai')
  → captureChub(url)
       1. URL에서 {author}/{slug} 추출
       2. fetch https://api.chub.ai/api/characters/{author}/{slug}?full=true
          (실패 시 avatars.charhub.io PNG → parsePngTavernCard 폴백)
       3. chara_card_v2.data → 매핑표대로 '번역 전' AssembledResult 조립
       4. translateCard(assembled) 로 번역 + 태그 정규화
       5. return { sections:[], title, imageUrl, assembledResult } 형태의 Captured
  → runImport(captured, url, userId)   ← 기존 파이프라인 그대로 Character+Collection 생성
```
국내 센터처럼 `assembledResult`를 직접 채워 classify 단계를 건너뜀(필드가 이미 구조화돼 있음).

### captureChub(url) 의사코드
```ts
export async function captureChub(url: string): Promise<Captured> {
  const { author, slug } = parseChubUrl(url)        // /characters/{author}/{slug}
  const card = await fetchChubCard(author, slug)    // full=true JSON, 실패 시 PNG 폴백
  if (!card.name) throw new Error('Chub 카드에 이름이 없습니다.')

  // 1) 번역 전 조립 (원문 그대로)
  const raw: AssembledCharacter = {
    name: card.name,                                 // ❌ 비번역
    gender: '',                                      // chara_card_v2엔 성별 없음
    tags: card.tags ?? [],
    additionalInfo: [card.description,
                     card.personality && `[성격]\n${card.personality}`,
                     card.creator_notes && `[제작자 메모]\n${card.creator_notes}`]
                    .filter(Boolean).join('\n\n'),
    openingMessage: card.first_mes ?? '',
    openingMessages: [card.first_mes, ...(card.alternate_greetings ?? [])]
                     .filter(Boolean)
                     .map((c, i) => ({ id: String(i), title: `도입부 ${i+1}`, content: c })),
    exampleDialogues: card.mes_example ?? '',
    avatarUrl,
  }
  const scenarioRaw = card.scenario ?? ''

  // 2) 번역 + 태그 정규화
  const { character, scenarioDescription } = await translateCard(raw, scenarioRaw)

  return {
    sections: [], title: character.name, imageUrl: avatarUrl ?? '',
    assembledResult: {
      characters: [character], scenarioDescription,
      tags: character.tags ?? [], title: character.name,
    },
  }
}
```

### 번역 모듈 (translate.ts)
**모델**: gemini-2.5-pro. `generateText()`가 현재 flash(`GEMINI_UTILITY_MODEL`) 고정이므로
→ `generateText`에 `model?` 파라미터 추가하고 `model = GEMINI_CHAT_MODEL`(=pro) 전달.
flash 버전 호출은 작성해두되 **주석 처리** + `// TODO: flash로 변경 예정`.

**호출 분리(토큰·트렁케이션 안전)**:
1. `translateShort()` — name 제외 짧은 필드(description, personality, scenario, creator_notes)를
   **JSON in → JSON out 1콜**로 묶어 번역(톤 일관성↑). 매크로 보존 지시 포함.
2. `translateNarrative()` — 긴 서술(openingMessages 각 항목, mes_example)은 **항목별 콜**(길이 보호).
3. `normalizeTags()` — `tagMap.ts` 로컬 매핑 우선 → 미정의 태그만 **1콜 배치 번역** → dedup·최대 15개.

**공통 번역 시스템 프롬프트 골자**:
```
너는 영문 롤플레이 캐릭터 카드를 자연스러운 한국어로 번역한다.
규칙:
- {{char}}, {{user}}, <START> 등 매크로/마커는 절대 번역·삭제하지 말고 원문 그대로 둔다.
- 고유명사(인명·지명)는 원문(영문) 그대로 유지한다.
- 의미·뉘앙스를 보존하고, 설명체가 아닌 자연스러운 번역체로.
- 입력 JSON의 키는 그대로, 값만 번역해 같은 JSON 구조로만 출력한다.
```

**translateCard 시그니처**:
```ts
async function translateCard(
  raw: AssembledCharacter, scenarioRaw: string
): Promise<{ character: AssembledCharacter; scenarioDescription: string }>
```

### route.ts 등록
```ts
if (matchesHost(url, 'chub.ai', 'characterhub.org')) {
  try { return NextResponse.json(await runImport(await captureChub(url.trim()), url.trim(), userId), { status: 201 }) }
  catch (e: any) { return NextResponse.json({ error: e.message ?? 'Chub 가져오기 실패' }, { status: 400 }) }
}
```
(zeta/melting/whif/tikita 분기 뒤, generic Tavern fetch 앞에 추가)

### 가이드/센터 노출 (CLAUDE.md 규칙)
- 가져오기 UI 센터 목록에 Chub 추가
- `app/(main)/guide/page.tsx`의 `FEATURE_SECTIONS`에 "외국 센터(Chub) 가져오기 + 자동 번역" 항목 추가

## 다음 단계
구현 착수: tavernCard 인터페이스 확장 → tagMap → translate.ts → chub.ts → route 등록 → 가이드/센터 노출
