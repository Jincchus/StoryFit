# Project Overview

## Table of Contents
1. [Project Definition](#project-definition)
2. [Core Concepts](#core-concepts)
3. [Version Roadmap](#version-roadmap)
4. [Architecture](#architecture)

---

## Project Definition

**StoryFit** — Novel-style roleplay AI chat service

Users create custom characters or select preset characters to engage in roleplay conversations with AI in a novel format (action lines in italic, dialogue in bold).

- **Platforms**: Web (Next.js) + Mobile (Expo)
- **AI**: Gemini / ChatGPT / Claude — switchable mid-conversation
- **Build system**: Turborepo monorepo

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Character** | AI persona. Composed of systemPrompt + scenarioDescription + exampleDialogues |
| **UserPersona** | User's alter-ego. Selectable per conversation |
| **Conversation** | A single chat room. Supports multiple characters |
| **Memory (3-tier)** | Short-term (last 10–15 messages) + Long-term (summaries per 10 messages) + Core (coreMemory + statusTimeline) |
| **Lorebook** | World-building entries injected into context when trigger keywords appear |
| **AIAdapter** | Abstracts Gemini / ChatGPT / Claude behind a single interface |

---

## Version Roadmap

| Version | Key Features |
|---------|--------------|
| **v1** | Basic roleplay, 3-tier memory, novel-style rendering, lorebook, regeneration, persona, stream abort, base rules block, generationConfig tuning |
| **v2** | Multi-user, character sharing, turn-order/tiki-taka mode, AI auto-coreMemory, Gemini context caching, message branching, token-based memory, avatar server upload |
| **v3** | Tavern Card import/export, multimodal, long-term memory RAG, Gemini Live voice roleplay |

---

## Architecture

```
StoryFit/ (Turborepo)
├── apps/
│   ├── web/          # Next.js 14 — Web UI + API Routes (backend)
│   └── mobile/       # Expo — iOS/Android
└── packages/
    └── shared/       # Shared types, API client
```

- Backend: Next.js API Routes (self-hosted)
- DB: PostgreSQL + Prisma ORM
- Auth: JWT — Access Token (1h) + Refresh Token (30d)
- AI API keys: server-side only
