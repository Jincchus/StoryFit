# DB Schema

## Table of Contents
1. [Tables](#tables)
2. [User](#user)
3. [Character](#character)
4. [UserPersona](#userpersona)
5. [Conversation](#conversation)
6. [ConversationCharacter](#conversationcharacter)
7. [Message](#message)
8. [Memory](#memory)
9. [Lorebook](#lorebook)
10. [Relationships](#relationships)

---

## Tables

| Table | Description |
|-------|-------------|
| User | User accounts |
| Character | AI characters (preset + custom) |
| UserPersona | User alter-egos |
| Conversation | Chat rooms |
| ConversationCharacter | Many-to-many mapping between Conversation and Character |
| Message | Chat messages |
| Memory | Long-term memory summaries |
| Lorebook | World-building dictionary entries |

---

## User

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| email | string (unique) | |
| passwordHash | string | |

---

## Character

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| name | string | |
| description | string | |
| systemPrompt | text | AI character instruction |
| scenarioDescription | text | Background scenario (separate from systemPrompt) |
| avatarUrl | string (nullable) | v1: external URL only |
| firstMessage | text | Opening message displayed on conversation start |
| alternateGreetings | string[] | Alternative opening message candidates (random pick) |
| exampleDialogues | text | few-shot example dialogues (2–3 examples) |
| tags | string[] | Genre/mood tags |
| safetyLevel | enum | strict \| standard \| relaxed |
| temperature | float | Default 0.9, range 0.0–2.0 |
| frequencyPenalty | float | Default 0.3 |
| presencePenalty | float | Default 0.3 |
| voiceConfig | JSON (nullable) | v3 Gemini Live voice settings |
| isPreset | bool | Whether this is a preset character |
| creatorId | string (nullable, FK→User) | null = preset; userId = user-owned |
| defaultAI | enum | gemini \| claude \| chatgpt |

---

## UserPersona

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| userId | string (FK→User) | |
| name | string | Name AI uses when addressing the user |
| description | text | Appearance, personality, background |
| additionalInfo | text | Extra settings (occupation, relationship, etc.) |

---

## Conversation

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| title | string | |
| summary | text | Preview text for home list |
| currentAI | enum | Currently active AI provider |
| userPersonaId | string (nullable, FK→UserPersona) | |
| coreMemory | text | Pinned facts that must never be forgotten |
| statusTimeline | text | Current episode state (always injected into prompt) |
| isSummarizing | bool | Flag to prevent concurrent memory summarization |

---

## ConversationCharacter

| Column | Type | Description |
|--------|------|-------------|
| conversationId | string (FK→Conversation) | |
| characterId | string (FK→Character) | |

PK: (conversationId, characterId)

---

## Message

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| conversationId | string (FK→Conversation) | |
| role | enum | user \| assistant |
| content | text | |
| aiModel | string | AI model used for this message |
| characterId | string (nullable, FK→Character) | Source character in multi-character responses |
| parentId | string (nullable, FK→Message) | v1: always null; branch tree activated in v2 |
| isSelected | bool | Whether this message is currently rendered (default true) |
| attachments | JSON[] | v3 multimodal attachments |

---

## Memory

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| conversationId | string (FK→Conversation) | |
| summary | text | AI-generated context summary |
| messageRangeStart | string (FK→Message) | Start of summarized message range |
| messageRangeEnd | string (FK→Message) | End of summarized message range |

---

## Lorebook

| Column | Type | Description |
|--------|------|-------------|
| id | string (PK) | |
| scope | enum | conversation \| character |
| scopeId | string | conversationId or characterId |
| keyword | string[] | Trigger keyword list |
| content | text | World-building description to inject |
| priority | int | Default 0; higher value injected first |
| scanDepth | int | How many recent messages to scan (default 5) |
| isEnabled | bool | |

---

## Relationships

| Relationship | Type |
|-------------|------|
| Character ↔ User | creatorId nullable — null for presets, userId for user-owned |
| Conversation ↔ Character | Many-to-many via ConversationCharacter mapping table |
| UserPersona ↔ User | Belongs to User |
| UserPersona ↔ Conversation | Optional reference from Conversation |
| Message.parentId | Always null in v1; no migration needed when branch tree is activated in v2 |
