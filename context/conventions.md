# Naming / Security / Git Conventions

## Table of Contents
1. [Naming Conventions](#naming-conventions)
2. [Security Policy](#security-policy)
3. [Git Strategy](#git-strategy)

---

## Naming Conventions

| Target | Rule | Example |
|--------|------|---------|
| File (component) | PascalCase | `ChatMessage.tsx` |
| File (util/hook) | camelCase | `useConversation.ts` |
| Variable / function | camelCase | `sendMessage`, `currentAI` |
| DB table | PascalCase (Prisma model) | `ConversationCharacter` |
| DB column | camelCase | `systemPrompt`, `creatorId` |
| API path | kebab-case | `/api/core-memory` |
| Enum value | lowercase | `gemini`, `claude`, `chatgpt` |
| Enum safetyLevel | lowercase | `strict`, `standard`, `relaxed` |

---

## Security Policy

- **AI API keys**: Server-side only (Next.js API Routes). Store in `.env`; never include in client bundle.
- **JWT**: Store Access Token in memory (or httpOnly cookie). Store Refresh Token in httpOnly cookie.
- **Input validation**: Validate request body and query params on every API endpoint.
- **Character ownership**: On PATCH/DELETE, verify `creatorId === currentUserId`. Presets (`creatorId === null`) must not be modified.
- **Conversation access control**: Verify ownership before reading or modifying a conversation.
- **.env files**: Never commit.

---

## Git Strategy

### Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feat/<description>` | `feat/lorebook-injection` |
| Bug fix | `fix/<description>` | `fix/stream-abort-save` |
| Refactor | `refactor/<description>` | `refactor/ai-adapter` |

### Commit Messages (Conventional Commits)

```
Feat: add a new feature
Fix: fix a bug
Refactor: improve code without changing behavior
Style: formatting, semicolons — no logic change
Docs: documentation changes
Test: add or update tests
Chore: build process, package updates
Core: changes to core logic or architecture
```

Example: `Feat: add JWT authentication for login`

### Release

- `main`: production deployment branch
- `develop`: integration branch (if used)
- Code review recommended before merging PRs
