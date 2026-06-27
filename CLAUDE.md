# StoryFit — Roleplay AI Chatbot

Novel-style roleplay AI chat service — parent repo with git submodule (apps/web), Next.js 14 (web + API) + Expo WebView wrapper (mobile).  
API: `http://localhost:3000/api` | Web: `http://localhost:3000` | Mobile: Expo Dev Client

## Rules (always follow)

- **AI key server isolation**: AI API keys must only be used in server-side code (Next.js API Routes). Never expose to client.
- **System prompt assembly order is fixed**: static prefix first — globalRules → personalRules → base rules → style → modeRules → UserPersona → character settings → scenario → openingScene → exampleDialogues — then volatile tail: statusTimeline → stats → inventory → Lorebook → long-term memory → coreMemory → closingRules. Static blocks MUST stay before volatile ones so the prefix hits Gemini implicit caching. Never reorder.
- **Lorebook token cap**: Sort matched entries by priority descending; exclude entries once cumulative total exceeds 1,000 tokens.
- **isSummarizing flag**: Prevent concurrent memory summarization — use `Conversation.isSummarizing` to block duplicate runs.
- **Partial stream save**: When SSE connection is aborted via AbortController, save the received partial content; discard only empty responses.
- **No frequencyPenalty range conversion**: Pass stored value directly to each adapter (default 0.3 is within safe range for both OpenAI and Gemini).
- **Claude adapter penalty unsupported**: Replace frequencyPenalty/presencePenalty with repetition-prevention instructions in the base rules block.
- **Regeneration behavior**: Delete the last assistant message then regenerate. User message edit deletes all subsequent messages.
- **Avatar image v1**: External URL input only. Server upload added in v2.
- **Message.parentId**: Fully active — branch create/switch/sibling navigation all implemented. parentId links messages into a tree; `isSelected` marks the active branch path.
- **사용자 기능 가이드 동기화**: 사용자가 체감할 수 있는 새 기능이나 설정을 추가/변경하면, `apps/web/app/(main)/guide/page.tsx`의 `FEATURE_SECTIONS`에도 해당 항목을 함께 추가/수정한다.
- **플레이스홀더 치환 필수**: 센터 상세 페이지에서 화면에 렌더링되는 **모든** 텍스트 필드(소개글·설명·캐릭터 설정·에피소드 제목·첫 장면 등)에 반드시 `replaceDisplayPlaceholders(text, userName, charNames)`를 적용한다. `charNames`는 `col.characters.map(c => c.name)` 전체 배열을 전달해야 한다(`{{char1}}`, `{{char2}}`… 모두 치환). 새 센터 페이지 작업 시, 그리고 기존 페이지 수정 시 모든 렌더 필드를 grep으로 점검해 누락을 검증한다: `grep -n "{[^}]*}" page.tsx | grep -v "replaceDisplay\|NovelText\|style\|className\|key\|src\|alt\|href\|onClick\|onChange\|disabled\|type\|placeholder"` — 남은 항목 중 사용자 입력 데이터가 포함된 것은 전부 치환 대상이다.
- **센터 공통 동작 ↔ 가이드 동기화**: 센터 공통 검색/필터/태그/페르소나 등 동작을 추가·변경하면 `context/adding-a-center.md`도 함께 갱신해, 새 센터 추가 시 누락 없이 전파되도록 한다.

## Novel-style Rendering

| Format | Rendering |
|--------|-----------|
| `*action*` | Light gray + italic |
| `"dialogue"` | White + bold |
| Plain text | Default style |

## Stack

| Layer | Tech |
|-------|------|
| Web frontend | Next.js 14, React |
| Mobile | Expo WebView wrapper (iOS/Android) — loads the deployed web app, no shared package |
| Backend | Next.js API Routes |
| DB | PostgreSQL + Prisma ORM |
| Auth | JWT (Access 1h + Refresh 30d) |
| AI | Gemini only — 채팅(스토리/멀티) `gemini-2.5-pro`, 백그라운드 유틸(요약·리캡·핵심메모리 압축) `gemini-2.5-flash`. Pro는 thinking을 끌 수 없어 `thinkingBudget` 0/미설정 시 동적(-1)으로 보정(`lib/ai/gemini.ts`). ⚠️ Pro는 무료 기간 한정(2026-10 종료 예정) — 종료 시 `GEMINI_CHAT_MODEL`을 flash로 복귀. Claude/ChatGPT는 "준비 중"(disabled), 어댑터 미구현; `streamChat` falls back to Gemini |

## Ports

| Service | Port |
|---------|------|
| Next.js (web + API) | 3000 |
| Expo Dev Server | 8081 |
| PostgreSQL | 5432 |

## Deploy

After modifying files in `apps/web`, always push in two steps:

```bash
# 1. Submodule (apps/web) — main branch
cd apps/web && git add <files> && git commit -m "..." && git push origin main

# 2. Update parent repo submodule pointer — master branch
cd ../.. && git add apps/web && git commit -m "Chore: apps/web 서브모듈 포인터 업데이트 (...)" && git push origin master
```

Server deploy command:
```bash
git pull origin master && git submodule update --remote apps/web && docker compose up --build -d
```

## Reference Docs

| Task | File |
|------|------|
| Project overview | `context/overview.md` |
| Frontend spec | `context/frontend-spec.md` |
| Backend spec | `context/backend-spec.md` |
| API endpoints | `context/endpoints.md` |
| DB schema | `context/db-schema.md` |
| Screen structure / routing | `context/frontend-structure.md` |
| Naming / security / Git conventions | `context/conventions.md` |
