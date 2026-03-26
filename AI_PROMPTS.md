# SuperMail AI — Prompts & Context Reference

All AI features use the Gemini API via `callGemini()` (`antymail.html:1761`). Config comes from `env.js` (`GEMINI_API_KEY`, `GEMINI_MODEL`).

---

## 1. AI Edit — Unified Compose/Reply/Rewrite (Cmd+A)

**Triggered by:** `Cmd+A` from anywhere in the app
**Function:** `openAIHover()` (line 1424) → `aiHoverGo()` (line 1497)
**Tools passed:** None — single-turn generation

### Behavior depends on context:

**A) Text selected (in compose body or reply textarea):**
```
Here is some text from an email:

"{selectedText}"

The user wants you to: {user's instruction}

Return ONLY the rewritten text. No explanation, no quotes, no preamble.
```
Replaces the selected text in-place with the result.

**B) In compose modal, no selection:**
```
To: {to}
Subject: {subject}

The user wants to write an email. Their instruction: {user's instruction}

Return ONLY the email body. No subject line unless asked.
```
Fills the compose body with the result.

**C) Replying to an email, no selection:**
```
Draft a reply to this email:
From: {selEmail.from} <{selEmail.fromEmail}>
Subject: {selEmail.subject}

{selEmail.preview}

The user's instruction: {user's instruction}

Return ONLY the reply body. No subject line, no "Dear...", unless the instruction asks for it.
```
Fills the reply textarea with the result.

**D) No compose/reply open:**
Opens compose first, then behaves like (B) above.

---

## 2. AI Chat Panel (line 2066)

**Triggered by:** User typing in the AI chat panel (opened via floating ✦ button or sidebar)
**Function:** `processAITurn()`
**Tools passed:** `GEMINI_TOOLS` (4 function declarations, line 1708)

### System Context (injected as first message every turn)

```
[System] You are an AI email assistant integrated into SuperMail, a Gmail client.
You help users search, read, and manage their emails.

Current user view:
- Folder: {folder}
- Filter tab: {cat}
- Search box: "{searchQ}"
- Emails loaded: {emails.length}
- Today: {YYYY-MM-DD}

IMPORTANT RULES:
1. When the user has an active search query or filter, ALWAYS prefer
   list_visible_emails first — it returns exactly what the user is already
   looking at. Only use search_emails if the user asks to search beyond
   what's currently visible, or if list_visible_emails doesn't have enough info.
2. When the user says "today", use the exact date ({YYYY-MM-DD}). "Today" means
   ONLY today, not yesterday. Use the structured after/before date params for
   precise filtering. For a single day: after=that day, before=next day.
3. When showing results, format them clearly.
4. Attachment cards with download buttons are rendered automatically from
   search_emails and get_email results. Do NOT call get_attachment unless the
   user explicitly asks to download a specific file.
5. Always use tools to access email data rather than guessing.
```

Dynamic values (computed at call time, not hardcoded):
- `folder` — current Gmail label (`INBOX`, `SENT`, etc.)
- `cat` — active filter tab (`all`, `unread`, `starred`, `attach`)
- `searchQ` — text in the search box
- `emails.length` — number of emails currently loaded
- Today's date via `new Date().toISOString().split('T')[0]`

### Conversation structure

```
contents = [
  systemContext,           // the [System] message above (role: "user")
  ...aiChatHistory         // full multi-turn history: user, model, function messages
]
```

The chat runs a **tool loop** (max 5 iterations per user message):
1. Send contents + tools to Gemini
2. If response contains `functionCall` parts → execute them → append `functionResponse` → loop back to step 1
3. If response is text only → done, render it
4. Attachment/email cards from tool results render **below** the model's text response

### Tools (line 1708)

#### `search_emails`
| Param | Type | Description |
|---|---|---|
| `query` | STRING (required) | Gmail search query WITHOUT date operators (e.g. `from:jph has:attachment`) |
| `after` | STRING | Only emails on or after this date, `YYYY-MM-DD`. For "today" use today's exact date. |
| `before` | STRING | Only emails strictly before this date, `YYYY-MM-DD`. Single day = after + next day as before. |
| `maxResults` | INTEGER | Max results, default 10, max 50 |
| `label` | STRING | Gmail label to search within (e.g. `INBOX`). Defaults to current folder. |

Date handling in execution (line 1785):
- `after`/`before` are converted to Gmail `after:YYYY/MM/DD` / `before:YYYY/MM/DD` query syntax
- Results are **post-filtered** by date as a safety net against Gmail timezone drift

**Returns:** `{ results: [...], query_used: "..." }`
Each result: `{ id, from, fromEmail, subject, date, snippet, hasAttachment, attachments: [{ filename, mimeType, size, attachmentId }] }`

#### `get_email`
| Param | Type | Description |
|---|---|---|
| `messageId` | STRING (required) | Gmail message ID |

**Returns:** `{ id, from, fromEmail, subject, date, body (plain text, max 3000 chars), attachments: [{ filename, mimeType, size, attachmentId }] }`

#### `get_attachment`
| Param | Type | Description |
|---|---|---|
| `messageId` | STRING (required) | Gmail message ID containing the attachment |
| `attachmentId` | STRING (required) | Attachment ID from `get_email` results |
| `filename` | STRING (required) | Filename |
| `mimeType` | STRING | MIME type |

**Returns:** `{ filename, status: "downloading", message }` — triggers actual browser download via `downloadAttachment()`.

**Note:** The AI is instructed NOT to call this unless the user explicitly asks to download. Attachment cards with download buttons are rendered automatically from `search_emails` / `get_email` results.

#### `list_visible_emails`
No parameters.

**Returns:** `{ folder, filter, searchQuery, count, emails: [{ id, from, fromEmail, subject, date, snippet, unread, starred, hasAttachment, attachments }] }`

Returns exactly what the user currently sees in the email list (respects folder, search box, and category filter). The AI is told to prefer this tool when the user has active filters.

---

## How `callGemini()` works (line 1761)

```
POST https://generativelanguage.googleapis.com/v1beta/models/{GEMINI_MODEL}:generateContent?key={GEMINI_API_KEY}

Body: { contents: [...], tools?: [...] }
```

- Returns `candidate.content` — a `{ role: "model", parts: [...] }` object
- Parts can be `{ text }` or `{ functionCall: { name, args } }`
- AI Edit extracts `.parts[0].text` (single-turn, no tools)
- AI Chat feeds function calls back through the tool loop

---

## Other features

### Pagination
- `loadEmails(append)` — when `append=true`, uses `nextPageToken` to fetch next page
- "Load older emails" button appears at bottom of email list when more pages exist
- `nextPageToken` resets on folder switch or refresh

### Attachment download
- `downloadAttachment(messageId, attachmentId, filename, mimeType)` (line 1115)
- Fetches binary via Gmail attachments API, decodes base64url, creates Blob + download link
- Triggered by clicking attachment chips in reader view or AI chat cards

### Multiple recipients
- To and Cc fields accept comma-separated emails (RFC822 `To:` header supports this natively)

---

## Where to modify

| What | File | Line |
|---|---|---|
| AI Edit hover (Cmd+A) | `antymail.html` | 1424–1555 |
| AI Edit prompt building | `antymail.html` | 1505–1525 |
| Chat system prompt | `antymail.html` | 2068 |
| Tool definitions | `antymail.html` | 1708–1757 |
| Tool execution | `antymail.html` | 1785–1845 |
| Gemini API call | `antymail.html` | 1761–1783 |
| API key / model | `env.js` | 3–4 |
