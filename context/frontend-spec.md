# Frontend Spec

## Table of Contents
1. [Novel-style Rendering](#novel-style-rendering)
2. [SSE Streaming](#sse-streaming)
3. [Multi-character @Mention](#multi-character-mention)
4. [Chat Screen Components](#chat-screen-components)
5. [State Management](#state-management)
6. [Character Creation Form Fields](#character-creation-form-fields)

---

## Novel-style Rendering

Implemented with a custom markdown parser (react-markdown + rehype plugin, or custom implementation).

| Format | Example | Rendering |
|--------|---------|-----------|
| `*action*` | `*she smiled shyly*` | Light gray + italic |
| `"dialogue"` | `"Long time no see"` | White + bold |
| Plain text | Narration | Default style |

---

## SSE Streaming

```
Event types:
  event: delta  →  data: { text }           # token chunk
  event: done   →  data: { messageId }      # stream complete
  event: error  →  data: { code, message }  # error
```

- Stream abort: close SSE connection via `AbortController`
- On abort: save received partial content; discard empty-only responses
- While generating: show abort button

---

## Multi-character @Mention

- `@CharacterName message` → only that character responds
- No mention → default (first) character responds
- Include `targetCharacterId?` in request body

---

## Chat Screen Components

```
Chat Screen
├── Top bar
│   ├── Current character info (avatarUrl + name)
│   └── AI selector dropdown (gemini | claude | chatgpt)
├── Message list
│   ├── Novel-style rendering (action italic, dialogue bold)
│   ├── AI model badge (per-message aiModel display)
│   └── Message actions: Regenerate / Edit / Delete
├── Streaming indicator + Abort button
├── Input field (supports multi-character @mention)
└── Conversation settings side panel
    ├── Persona selector
    ├── Core memory editor
    ├── statusTimeline editor
    └── Lorebook manager (keyword · content · priority · scan depth)
```

---

## State Management

- On conversation entry: display `firstMessage` automatically (no API call)
- `alternateGreetings[]` — randomly select one when starting a conversation (no API call)
- Message list: cursor pagination (`?cursor=<messageId>&limit=30`)
- On AI change: `PATCH /api/conversations/:id/ai` → context reassembled

---

## Character Creation Form Fields

| Field | Type | Description |
|-------|------|-------------|
| name | string | Character name |
| description | string | Character description |
| systemPrompt | string | AI instruction text |
| scenarioDescription | string | World/background description |
| avatarUrl | string | External image URL (v1) |
| firstMessage | string | Auto-displayed opening message |
| alternateGreetings[] | string[] | Alternative opening message candidates |
| exampleDialogues | string | few-shot example dialogues |
| tags[] | string[] | Genre/mood tags |
| safetyLevel | strict\|standard\|relaxed | AI safety level |
| temperature | float (0.0–2.0) | Creativity (default 0.9) |
| frequencyPenalty | float (default 0.3) | Word repetition suppression |
| presencePenalty | float (default 0.3) | Topic repetition suppression |
| defaultAI | gemini\|claude\|chatgpt | Default AI provider |
