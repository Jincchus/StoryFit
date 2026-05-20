# Screen Structure / Routing

## Table of Contents
1. [Folder Structure](#folder-structure)
2. [Screen List](#screen-list)
3. [Screen Flow](#screen-flow)

---

## Folder Structure

```
apps/
├── web/                        # Next.js 14
│   ├── app/                    # App Router
│   │   ├── (auth)/
│   │   │   └── login/
│   │   ├── (main)/
│   │   │   ├── page.tsx        # Home — conversation list
│   │   │   ├── characters/     # Character list + creation
│   │   │   ├── personas/       # My persona management
│   │   │   └── conversations/
│   │   │       └── [id]/       # Chat screen
│   │   └── api/                # API Routes (backend)
│   │       ├── auth/
│   │       ├── characters/
│   │       ├── personas/
│   │       ├── conversations/
│   │       ├── chat/
│   │       ├── messages/
│   │       └── lorebooks/
│   └── components/
│       ├── chat/               # Chat-related components
│       ├── character/          # Character-related components
│       └── ui/                 # Shared UI
└── mobile/                     # Expo
    └── app/                    # Expo Router (file-based routing)
        ├── (auth)/
        ├── (tabs)/
        │   ├── index.tsx       # Home
        │   ├── characters.tsx
        │   └── personas.tsx
        └── conversations/
            └── [id].tsx        # Chat screen

packages/
└── shared/
    ├── types/                  # Shared type definitions
    └── api/                    # API client
```

---

## Screen List

| Screen | Path (web) | Description |
|--------|-----------|-------------|
| Login | `/login` | Email/password login |
| Home | `/` | Conversation list (summary preview) + new conversation button |
| Character list | `/characters` | Presets + custom list, tag filter, avatarUrl + firstMessage preview |
| Character creation | `/characters/new` | Custom character creation form |
| Character edit | `/characters/:id/edit` | Edit custom character |
| Persona management | `/personas` | UserPersona list + create/edit/delete |
| Chat | `/conversations/:id` | Chat screen + conversation settings side panel |

---

## Screen Flow

```
Login
  ↓
Home (conversation list)
  ├── New conversation → Select character → Select persona (optional) → Select AI → Chat
  └── Existing conversation → Chat

Chat screen
  ├── Top: AI switch dropdown
  ├── Messages: Regenerate / Edit / Delete
  └── Side panel: Persona · Core memory · Timeline · Lorebook management
```
