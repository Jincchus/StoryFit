# Implementation Plan: Zeta, Whif, Melting Integrations Refactoring

This plan details the changes required to polish and extend features for Zeta, Whif, and Melting integrations. It addresses interface refinements, scraping enhancements (especially Zeta lorebooks), placeholder replacements, custom character creation, and ongoing conversation warnings.

---

## Proposed Changes

### Zeta Integration Refinement

#### [MODIFY] [zeta page.tsx](file:///C:/StoryFit/apps/web/app/(zeta)/zeta/page.tsx)
- Exclude the interaction count badge (`💬 1XX만`) from the character cards.
- Add support for manually creating a plot:
  - Add a `+ 새 플롯 만들기` modal button in the header menu.
  - POST to `/api/collections` with `title` and a virtual URL matching `https://zeta-ai.io/local/${Date.now()}`.

#### [MODIFY] [zeta plots [id] page.tsx](file:///C:/StoryFit/apps/web/app/(zeta)/zeta/plots/[id]/page.tsx)
- Exclude the interaction count badge (`💬 1XX만`) from the page header.
- Exclude the cover username display (`@username`) from the cover.
- Exclude the "크리에이터" (Creator) section.
- Query real database lorebooks using `api.get('/api/lorebooks?collectionId=' + id)` and display them with an expand/collapse toggle.
- When clicking "새로운 대화 시작하기", check if `existingConvs.length > 0`. If true, show a custom `ConfirmDialog` warning the user that an ongoing chat already exists before launching the persona selection.
- Add manual character management support:
  - Add a `+ 직접 캐릭터 등록` button to redirect to `/characters/new?isZeta=true&collectionId=${col.id}`.
  - Add `✏ 수정` and `✕ 삭제` buttons next to characters inside the character list, redirecting to `/characters/${c.id}/edit?isZeta=true` for edit.

---

### Zeta Lorebook Scraping & Import

#### [MODIFY] [capture.ts](file:///C:/StoryFit/apps/web/lib/import/capture.ts)
- In `captureZeta(url)`:
  - Iterate over the scraped `loreUrls` (Zeta lorebook links).
  - For each lorebook URL, fetch the HTML page.
  - Preprocess the text using `preprocessZetaText(loreHtml)`.
  - Call Gemini API (`generateText`) with a prompt that parses the text into keywords and content JSON.
  - Collect all parsed entries and store them in `captured.lorebooks`.

#### [MODIFY] [import route.ts](file:///C:/StoryFit/apps/web/app/api/characters/import/route.ts)
- In `runImport`:
  - **Remove** the creation of the initial conversation and first message on URL import.
  - Create the `CharacterCollection` record directly with `coverImageUrl` and metadata.
  - If `captured.lorebooks` has entries, create them as `Lorebook` records in the database with `scope: 'collection', scopeId: collection.id`. They will be automatically cloned to any new conversations created from this collection.
  - Return `{ characterId, collectionId }` to the client.

---

### Whif Placeholder Substitution

#### [MODIFY] [conversations [id] route.ts](file:///C:/StoryFit/apps/web/app/api/conversations/[id]/route.ts)
- Include the `user` relation in `prisma.conversation.findUnique` to fetch `displayName`.
- Fall back dynamically for the `personaName`:
  `const personaName = conv.personaCharacter?.name || conv.user?.displayName || '나'`
- Perform `replacePlaceholders` substitution on `conv.messages` even if `personaCharacter` is not set yet, so that placeholders like `{{user}}` and `{{char}}` are rendered correctly in the UI.

#### [MODIFY] [chat route.ts](file:///C:/StoryFit/apps/web/app/api/conversations/[id]/chat/route.ts)
- Include the `user` relation in `prisma.conversation.findUnique`.
- Retrieve `personaName = conv.personaCharacter?.name || conv.user?.displayName || '나'`.
- Substitute `{{user}}` and `{{char}}` placeholders in the message contents of the history array *before* feeding it into the LLM history. This ensures the LLM sees the resolved names.

---

### Melting Integration Refinement

#### [MODIFY] [capture.ts](file:///C:/StoryFit/apps/web/lib/import/capture.ts)
- In `captureMelting`:
  - Check for multiple candidate keys in `apiData.bot` to extract detailed properties:
    - First scene/opening message: `opening`, `firstMessage`, `greeting`, `intro`, `introduction`, `firstScene`, `openingMessage`, `startMessage`
    - Detailed profile: `profile`, `characterProfile`, `profileDescription`, `detailedDescription`
    - Timeline: `timeline`, `characterTimeline`, `timelineDescription`, `timelineSetting`
    - User setting: `userSetting`, `userSettings`, `userPersonaSetting`, `userPersonaSettings`, `userPersona`, `userDefaultSetting`, `userDefaultSettings`, `userPersonaDescription`
  - Append profile, timeline, and user settings blocks to `additionalInfo` in a clear format (e.g. `[캐릭터 프로필]`, `[타임라인]`, `[유저 기본 설정]`).
  - In the web scraping fallback (if API interception fails), ensure both "상세 설명" and "첫 장면" tabs are clicked and scraped.

#### [MODIFY] [melting page.tsx](file:///C:/StoryFit/apps/web/app/(melting)/melting/page.tsx)
- Add support for manually creating a character:
  - Add a `+ 새 캐릭터 만들기` modal button in the header.
  - POST to `/api/collections` with `title` and a virtual URL matching `https://melting.chat/local/${Date.now()}`.

#### [MODIFY] [melting characters [id] page.tsx](file:///C:/StoryFit/apps/web/app/(melting)/melting/characters/[id]/page.tsx)
- Parse `userSettings` from `col.meltingMeta` or by extracting the `[유저 기본 설정]` block from `mainChar.additionalInfo` using a regex match.
- Pass `userSettings` to `WhifPersonaModal` as `defaultSettings={userSettings}` so that it populates the character persona creation settings.
- Add "진행 중인 대화" (Ongoing conversations) section below details.
- Query ongoing conversations using `api.get('/api/conversations?characterId=' + mainChar.id)` and store in `existingConvs`.
- When clicking "대화 시작하기", check if `existingConvs.length > 0`. If true, show a custom `ConfirmDialog` warning the user before showing the persona modal.
- Add `✏ 수정` button redirecting to `/characters/${mainChar.id}/edit?isMelting=true`.

---

### Common Page Updates

#### [MODIFY] [characters new page.tsx](file:///C:/StoryFit/apps/web/app/(main)/characters/new/page.tsx)
- Support query parameters `isZeta` and `isMelting`.
- Filter collection options and redirect back to `/zeta` or `/melting` after creation.

#### [MODIFY] [characters [id] edit page.tsx](file:///C:/StoryFit/apps/web/app/(main)/characters/[id]/edit/page.tsx)
- Support query parameters `isZeta` and `isMelting`.
- Filter collection options and redirect back to `/zeta` or `/melting` after update.

#### [MODIFY] [whif page.tsx](file:///C:/StoryFit/apps/web/app/(main)/whif/page.tsx)
- For universe chat starting: check if `chats` contains an ongoing conversation with the same characters. If so, display a `ConfirmDialog` warning before starting a new chat room.
- For character 1:1 chat starting: check if `chats` contains a 1:1 conversation with this character. If so, display a `ConfirmDialog` warning.

---

## Verification Plan

### Automated Tests
- Run Next.js build: `npm run build` or equivalent.
- Run database migrations: `npx prisma db push` or similar to confirm schema consistency.

### Manual Verification
1. **Zeta**: Import a Zeta URL with a lorebook. Check that:
   - No usage count or creator is visible.
   - Lorebooks are imported and displayed on the plot page with expand/collapse buttons.
   - Starting a chat warns if there's an ongoing chat.
   - When entering the chat, the lorebook is correctly copied and active.
2. **Whif**: Start a chat room. Verify that the message history correctly replaces `{{user}}` with the user's name/nickname.
3. **Melting**: Import a Melting URL. Verify that:
   - "첫 장면" is correctly imported and saved.
   - Profile, timeline, and user settings are populated in details.
   - Starting a chat carries over the user's default setting to the persona creation modal.
4. **Custom creation**: Manually create a plot/character in Zeta/Melting and check that you can add characters, edit them, and start chats.
