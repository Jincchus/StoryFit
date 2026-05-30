# StoryFit — Roleplay AI Chatbot

Novel-style roleplay AI chat service — Turborepo monorepo, Next.js 14 (web + API) + Expo (mobile).  
API: `http://localhost:3000/api` | Web: `http://localhost:3000` | Mobile: Expo Dev Client

## Rules (always follow)

- **AI key server isolation**: AI API keys must only be used in server-side code (Next.js API Routes). Never expose to client.
- **System prompt assembly order is fixed**: Base rules(0) → UserPersona(1) → coreMemory(2) → statusTimeline(3) → Character systemPrompt+scenarioDescription(4) → exampleDialogues(5) → Lorebook(6) → long-term memory(7) → recent messages(8). Never reorder.
- **Lorebook token cap**: Sort matched entries by priority descending; exclude entries once cumulative total exceeds 1,000 tokens.
- **isSummarizing flag**: Prevent concurrent memory summarization — use `Conversation.isSummarizing` to block duplicate runs.
- **Partial stream save**: When SSE connection is aborted via AbortController, save the received partial content; discard only empty responses.
- **No frequencyPenalty range conversion**: Pass stored value directly to each adapter (default 0.3 is within safe range for both OpenAI and Gemini).
- **Claude adapter penalty unsupported**: Replace frequencyPenalty/presencePenalty with repetition-prevention instructions in the base rules block.
- **Regeneration behavior**: Delete the last assistant message then regenerate. User message edit deletes all subsequent messages.
- **Avatar image v1**: External URL input only. Server upload added in v2.
- **Message.parentId v1**: Always null. Branch tree activated in v2.

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
| Mobile | Expo (iOS/Android) |
| Shared package | packages/shared (types, API client) |
| Backend | Next.js API Routes |
| DB | PostgreSQL + Prisma ORM |
| Auth | JWT (Access 1h + Refresh 30d) |
| AI | Gemini / ChatGPT / Claude (adapter pattern) |
| Build | Turborepo |

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
