# Book Editor — AI Features PRD
**Phase 4: AI-Powered Writing Assistance**
Version 1.0 · 2026-04-29

---

## Overview

The existing editor (Phases 1–3) passively flags prose problems with colour highlights. Phase 4 makes those flags actionable by wiring three AI features into the single-file app:

| Feature | What it does |
|---------|-------------|
| **Inline Rewrite Popovers** | Click any `<mark>` span → popover shows 2 AI-generated rewrites for that sentence |
| **AI Chat Sidebar** | Persistent panel: ask free-form questions or give instructions scoped to the active chapter |
| **Chapter Critique** | On-demand button → one structured critique of the full chapter's prose weaknesses |

All three share a single AI API key stored in `localStorage`. No backend, no build step — still one `index.html`.

---

## Constraints (carry-forward from Phases 1–3)

- Single `index.html`, no CDN, no module bundler.
- All state in `localStorage`.
- `type="module"` script block.
- New JS sections (≤ ~350 lines total across all three features) are appended as clearly labelled inline sections, consistent with existing pattern.
- No changes to existing Phase 1–3 logic except the minimum necessary wiring.

---

## Prerequisites: API Key Settings Modal

All three features require the user's AI API key.

### UX
- Gear icon (⚙) in the toolbar, right side.
- Opens a modal with:
  - `<input type="password">` — "AI API Key" (placeholder: `Your API key…`)
  - "Save" button → stores key in `localStorage` under `book-api-key`
  - "Remove" link → clears key, disables AI features
  - One-line status: "Connected" or "No key saved"
- Key is never sent anywhere except the configured AI API endpoint.

### Validation
- On Save: fire a minimal `/v1/messages` call (1-token ping). Show "Verified ✓" or the API error message.
- If key is absent when any AI feature is triggered, open the settings modal automatically with the message "Add your API key to use AI features."

### Storage key
`book-api-key` in `localStorage`.

---

## Feature 1 — Inline Rewrite Popovers

### Trigger
`click` on any `<mark>` element inside `#editor`.

### Popover UI
- Floats near the clicked span (positioned via `getBoundingClientRect`).
- Header: highlight type label (e.g. "Very Hard Sentence") + ✕ close button.
- Body: spinner while loading, then two `<button>` rewrite options labelled "Option A" and "Option B".
- Footer: "Accept" (replaces original sentence in editor), "Try again" (re-calls API), "Dismiss".
- Only one popover open at a time; new click closes previous.
- `Escape` closes.

### API Call
Model: fast model (`AI_MODEL_FAST`)

System prompt (constant):
```
You are a prose editor. Rewrite the given sentence to fix the flagged issue.
Return exactly two alternatives separated by the token |||.
No preamble, no labels, no explanation.
```

User message template:
```
Issue: {highlightType}
Sentence: {sentenceText}
```

Where `{highlightType}` is one of: "very hard sentence", "hard sentence", "wordy phrase", "passive voice", "adverb overuse".

### Accept behaviour
- Replace the sentence containing the clicked `<mark>` span with the chosen rewrite text.
- Call `saveCurrentContent()` and `runHighlights()` immediately after.
- Cursor moves to end of replaced sentence.

### Error handling
- HTTP error or network failure: show error message in popover body, "Try again" button visible.
- Timeout (10 s): treat as error.

### Exit criteria
- Clicking a red `<mark>` span opens the popover with the correct type label.
- Two rewrite options appear after ≤ 3 s on a normal connection.
- Accepting Option A replaces the original sentence in the editor without changing any other content.
- Highlights re-run after acceptance; popover is gone.
- Pressing ✕ or Escape closes the popover.

---

## Feature 2 — AI Chat Sidebar

### Layout
- New right-hand panel, 280 px wide, replaces the current stats panel **only when toggled open**. Stats panel slides back when chat is closed. On narrow viewports (< 900 px), chat overlays the editor as a drawer.
- Toggle: "Chat" button in toolbar (right side, next to ⚙).
- Panel persists open/closed state in `localStorage` under `book-chat-open`.

### Chat history
- Messages stored in `localStorage` under `book-chat-{chapterId}` as a JSON array `[{role, content, ts}]`.
- History is chapter-scoped: switching chapters shows that chapter's history (or empty).
- Max 40 messages stored per chapter (oldest pruned first).
- "Clear" icon (🗑) in panel header clears current chapter's history.

### Input area
- `<textarea>` at bottom of panel, auto-growing (max 6 lines).
- Send on `Enter` (not `Shift+Enter`); `Shift+Enter` = newline.
- Submit button for mouse users.

### API Call
Model: fast model (`AI_MODEL_FAST`)

System prompt (injected fresh each call, not stored in history sent to API):
```
You are a writing coach helping improve a work-in-progress book chapter.
The current chapter text is provided below. Answer questions and give
advice about this specific text. Be concise and specific.

--- CHAPTER TEXT ---
{chapterPlainText}
```

Conversation history: last 10 `{role, content}` pairs sent as the `messages` array.

User message: the textarea content.

### Streaming
Use the streaming API (`"stream": true`) — assistant response streams into the panel as it arrives, token by token.

### UX details
- User messages: right-aligned, light blue background.
- Assistant messages: left-aligned, white background, markdown rendered (bold/italic/lists only — no code blocks needed).
- Scroll to bottom on new message.
- "Stop" button appears while streaming; clicking it aborts the fetch.

### Exit criteria
- Asking "What is the reading grade of this chapter?" returns a response that references the chapter content.
- Sending a message while streaming is disabled (input locked until response completes or stopped).
- Closing and reopening the panel shows the previous conversation.
- Switching chapters shows that chapter's history.
- "Clear" removes all messages from view and localStorage for that chapter.

---

## Feature 3 — Chapter Critique

### Trigger
"Critique" button in the stats panel (below existing stats, above highlight toggle).

### Modal UI
- Full-screen overlay modal, scrollable.
- Header: "Chapter Critique — {chapter title}" + ✕ close.
- Body: streams the critique in as it arrives (same pattern as chat).
- Footer: "Copy to clipboard" button.
- `Escape` closes (with confirmation if streaming in progress).

### API Call
Model: quality model (`AI_MODEL_QUALITY`, better reasoning for holistic critique)

System prompt:
```
You are a developmental editor reviewing a book chapter draft.
Analyse the prose and provide structured feedback covering:
1. Sentence variety and rhythm
2. Passive voice and weak verb choices
3. Adverb and filler-word overuse
4. Paragraph pacing and transitions
5. One specific paragraph to prioritize for revision (quote it)

Be direct and specific. Use the actual text in your examples.
Maximum 400 words.
```

User message:
```
{chapterPlainText}
```

### Caching
- After a successful critique, store result in `localStorage` under `book-critique-{chapterId}`.
- When the modal reopens, show cached result immediately with a "Regenerate" button.
- Cache is invalidated when chapter word count changes by ≥ 50 words.

### Exit criteria
- Clicking "Critique" on the 500-word sample chapter returns structured feedback within 10 s.
- Feedback references specific sentences from the chapter text.
- Closing and reopening the modal shows the cached result (not a spinner).
- "Regenerate" triggers a fresh API call.

---

## Shared Infrastructure

### `ai.js` section (new, ~100 lines)
Single module section handling all API calls:

```js
// ai.js
const AI_URL           = '<configured AI API endpoint>';
const AI_VERSION       = '<API version>';
const AI_MODEL_FAST    = '<fast model ID>';
const AI_MODEL_QUALITY = '<quality model ID>';

function getKey() { return localStorage.getItem('book-api-key') || ''; }

async function callAI({ model, system, messages, signal }) { … }
async function* streamAI({ model, system, messages, signal }) { … }
```

All three features call into `ai.js`. No feature imports directly from another feature.

### Rate limiting / error UX
- 429: show "Rate limit reached — wait a moment and try again."
- 401: show "Invalid API key — check Settings."
- Network error: show "Could not reach the AI service — check your connection."
- All error messages appear inline (in the popover, chat bubble, or critique body) — no `alert()`.

---

## Scope Boundaries (what Phase 4 does NOT include)

- No user accounts or server-side storage.
- No model selector UI (models are hardcoded per feature as above).
- No token usage / cost display.
- No fine-tuning or custom prompts UI.
- No image support.
- No multi-file export of chat history.

---

## Implementation Plan

| Step | Deliverable | Lines est. |
|------|-------------|-----------|
| 4.0 | API key settings modal + `ai.js` section | ~120 |
| 4.1 | Inline rewrite popovers | ~130 |
| 4.2 | AI chat sidebar (layout + logic + streaming) | ~200 |
| 4.3 | Chapter critique modal | ~80 |
| 4.4 | Wire-up, CSS tokens, accessibility pass | ~40 |

Total estimated addition: ~570 lines to existing ~1370-line file.

---

## Acceptance Test (full Phase 4)

Using the Chapter 1 sample text (~500 words):

1. Enter a valid API key in Settings → status shows "Connected".
2. Click a red `<mark>` → popover appears with two rewrites → accept one → sentence updates, highlights re-run.
3. Open Chat → ask "Which paragraph has the most passive voice?" → response references a specific paragraph.
4. Close and reopen Chat → history preserved.
5. Click "Critique" → streaming feedback appears → references real sentences → close → reopen → cached result shown.
6. Remove API key → clicking any AI feature opens Settings modal automatically.
