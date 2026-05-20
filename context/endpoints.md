# API Endpoints

## Table of Contents
1. [Auth](#auth)
2. [Characters](#characters)
3. [Personas](#personas)
4. [Conversations](#conversations)
5. [Messages / Chat](#messages--chat)
6. [Lorebooks](#lorebooks)

---

## Auth

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/login` | Login (email, password) |
| POST | `/api/auth/logout` | Logout (invalidate Refresh Token) |
| POST | `/api/auth/refresh` | Issue new Access Token using Refresh Token |

---

## Characters

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/characters` | List presets + user's custom characters (tag filter: `?tags=`) |
| POST | `/api/characters` | Create custom character |
| GET | `/api/characters/:id` | Get character detail |
| PATCH | `/api/characters/:id` | Update character |
| DELETE | `/api/characters/:id` | Delete character |

---

## Personas

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/personas` | List user's personas |
| POST | `/api/personas` | Create persona |
| PATCH | `/api/personas/:id` | Update persona |
| DELETE | `/api/personas/:id` | Delete persona |

---

## Conversations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/conversations` | List conversations |
| POST | `/api/conversations` | Create new conversation |
| GET | `/api/conversations/:id` | Get conversation detail (includes characters, persona, AI settings) |
| DELETE | `/api/conversations/:id` | Delete conversation |
| PATCH | `/api/conversations/:id/ai` | Switch AI provider (reassembles context) |
| PATCH | `/api/conversations/:id/core-memory` | Manually edit core memory |
| PATCH | `/api/conversations/:id/status-timeline` | Edit timeline status |

### POST /api/conversations Request Body

```json
{
  "characterIds": ["string"],
  "userPersonaId": "string (optional)",
  "defaultAI": "gemini | claude | chatgpt",
  "title": "string (optional)"
}
```

---

## Messages / Chat

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/conversations/:id/messages` | List messages (cursor pagination) |
| POST | `/api/chat` | Send message (SSE streaming) |
| POST | `/api/conversations/:id/regenerate` | Regenerate last response (SSE) |
| PATCH | `/api/messages/:id` | Edit user message → delete subsequent + regenerate |
| DELETE | `/api/messages/:id` | Delete single message |

### GET /api/conversations/:id/messages Query

```
?cursor=<messageId>&limit=30
```

### POST /api/chat Request Body

```json
{
  "conversationId": "string",
  "content": "string",
  "targetCharacterId": "string (optional, multi-character mention)"
}
```

### SSE Response Events

```
event: delta  →  data: { "text": "..." }
event: done   →  data: { "messageId": "..." }
event: error  →  data: { "code": "...", "message": "..." }
```

---

## Lorebooks

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/lorebooks` | List lorebook entries (`?scope=conversation&scopeId=:id` or `?scope=character&scopeId=:id`) |
| POST | `/api/lorebooks` | Create lorebook entry |
| PATCH | `/api/lorebooks/:id` | Update lorebook entry |
| DELETE | `/api/lorebooks/:id` | Delete lorebook entry |
