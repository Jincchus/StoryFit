# Task Checklist

## 1. ZETA Integration Updates
- `[x]` Remove usage count from Zeta card list page
- `[x]` Remove usage count, creator cover info, and creator section from Zeta detail page
- `[x]` Scrape Zeta lorebooks during import in `captureZeta`
- `[x]` Parse Zeta lorebooks using Gemini in `captureZeta`
- `[x]` Display imported lorebooks dynamically on Zeta detail page
- `[x]` Add custom plot creation support to Zeta list page
- `[x]` Add manual character creation and edit redirects to Zeta detail page

## 2. WHIF Placeholder & Chat Warning Updates
- `[x]` Include user displayName in conversation query
- `[x]` Dynamically substitute placeholders `{{user}}`, `{{char}}` in `conversations/[id]` API
- `[x]` Substitute placeholders in conversation message history *before* sending to LLM
- `[x]` Add ongoing chat warnings when starting a chat in WHIF Center

## 3. MELTING Integration Updates
- `[x]` Ensure first scene is imported correctly in `captureMelting`
- `[x]` Capture and format detailed profile, timeline, and user settings in `captureMelting`
- `[x]` Parse user settings and pass as `defaultSettings` to `WhifPersonaModal` in Melting detail page
- `[x]` Query and display ongoing conversations on Melting detail page
- `[x]` Add ongoing chat warnings when starting a chat on Melting detail page
- `[x]` Add manual character creation support in Melting page
- `[x]` Add edit character redirect support on Melting detail page

## 4. COMMON Updates
- `[x]` Support query parameters `isZeta` and `isMelting` in character new/edit pages
- `[x]` Remove initial conversation/message auto-creation on URL import in `runImport`
- `[x]` Verify everything builds successfully
