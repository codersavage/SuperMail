# SuperMail Agent Integrations — Implementation Guide

## Architecture Overview

SuperMail routes messages to AI agents by detecting special recipients in the To field. When a user "forwards" or "replies" to an agent name (e.g., "Gemini", "Claude", "Notion"), the system intercepts the send, routes the email thread + user message to the appropriate backend, and saves the response as a real email in the thread.

### Current State
- One hardcoded agent: **Gemini** (matched by `gemini` in the To field)
- Gemini responses are sent as real emails to the user, threaded into the conversation
- Uses the existing Gemini API key (same one powering the AI chat panel)

### Target State
- A registry of agent connectors (`AGENTS` object)
- Each agent is a simple config: name, icon, match pattern, execute function
- Adding a new agent = adding one entry to the registry + providing credentials
- Settings UI for managing API keys and toggling agents on/off

---

## Step 1: Refactor into Agent Registry

**Goal:** Extract the hardcoded Gemini intercept into a pluggable registry.

1. Create an `AGENTS` object at the top of the JS section:
   ```js
   const AGENTS = {
     gemini: {
       name: 'Gemini',
       icon: '✦',
       match: (to) => /^gemini(@|$)/i.test(to),
       execute: async (threadContext, userMessage) => {
         // existing Gemini callGemini() logic moves here
         return { text: '...' };
       }
     }
   };
   ```

2. Replace the `isGeminiRecipient()` check with a generic `findAgent(to)` that iterates the registry.

3. Replace the Gemini-specific send logic in `sendReply()` with:
   ```js
   const agent = findAgent(to);
   if (agent) { await handleAgentSend(agent, ...); return; }
   ```

4. Extract `handleAgentSend(agent, threadMessages, userMessage, selEmail)` as a shared function that:
   - Builds thread context
   - Calls `agent.execute()`
   - Sends the response as a real email to self
   - Reloads the thread

**Files changed:** `index.html` only (all client-side)
**Dependencies:** None
**Auth:** Uses existing credentials

---

## Step 2: Add Claude as a Second Agent

**Goal:** Prove the registry works with a second AI agent.

1. Add Claude to the registry:
   ```js
   claude: {
     name: 'Claude',
     icon: '◈',
     match: (to) => /^claude(@|$)/i.test(to),
     execute: async (threadContext, userMessage) => {
       // Call Anthropic API
     }
   }
   ```

2. The Anthropic API requires a server-side proxy (API key can't be exposed in the browser). Options:
   - **Netlify function** (like the existing `netlify/functions/gemini.js`) — add a `netlify/functions/claude.js`
   - Or add a generic `netlify/functions/agent-proxy.js` that accepts a `provider` param

3. Store the Claude API key in Netlify environment variables (same pattern as Gemini).

4. The execute function calls `/api/claude` with the thread context + message.

**Files changed:** `index.html`, new `netlify/functions/claude.js`
**Dependencies:** `@anthropic-ai/sdk` (or raw fetch to `https://api.anthropic.com/v1/messages`)
**Auth:** Anthropic API key in env vars

---

## Step 3: Settings UI for API Keys

**Goal:** Let users configure which agents are active and provide their own API keys.

1. Add a settings panel (gear icon in sidebar or command palette).

2. For each agent in the registry, show:
   - Toggle on/off
   - API key input (stored in localStorage, encrypted or at least not in plain sessionStorage)
   - Test button (sends a simple prompt, verifies the key works)

3. Update `findAgent()` to skip disabled agents.

4. Update the Gemini badge to show whichever agent is matched (show the agent's icon + name).

**Files changed:** `index.html` (settings panel HTML/CSS/JS)
**Dependencies:** None
**Auth:** User-provided API keys stored in localStorage

---

## Step 4: Action-Oriented Agents (Notion, Calendar, Linear, Slack)

**Goal:** Agents that don't just reply with text but actually perform actions in external services.

These are fundamentally different from AI agents — they take the email context and **do something** with it. The execute function returns both a text response (for the email thread) and performs a side effect.

### Notion
- **What it does:** "Forward to Notion" creates a page in a Notion database with the email content, or appends to an existing page.
- **API:** Notion API (`https://api.notion.com/v1/pages`, `/v1/blocks`)
- **Auth:** Notion integration token (user creates an internal integration at notion.so/my-integrations)
- **Setup steps:**
  1. User creates a Notion integration and gets an API token
  2. User shares the target database/page with the integration
  3. In SuperMail settings, user pastes the token and selects a default database
  4. Optionally, the agent can use Gemini/Claude to decide how to structure the content before pushing to Notion
- **Proxy needed:** Yes — Notion API key can't be in the browser. Add `netlify/functions/notion.js`

### Google Calendar
- **What it does:** "Forward to Calendar" extracts event details from the email and creates a calendar event.
- **API:** Google Calendar API (`https://www.googleapis.com/calendar/v3/calendars/primary/events`)
- **Auth:** Same Google OAuth — just add `https://www.googleapis.com/auth/calendar.events` to `GMAIL_SCOPES`
- **Setup steps:**
  1. Add the calendar scope to the OAuth consent screen
  2. Add the scope to `GMAIL_SCOPES` in `index.html`
  3. Users will need to re-authenticate (new scope prompt)
  4. The agent uses Gemini to extract date/time/title/description from the email, then creates the event
- **Proxy needed:** No — uses the existing Google OAuth token

### Linear
- **What it does:** "Forward to Linear" creates an issue from the email content.
- **API:** Linear GraphQL API (`https://api.linear.app/graphql`)
- **Auth:** Linear API key (user generates at linear.app/settings/api)
- **Setup steps:**
  1. User generates a Linear personal API key
  2. In SuperMail settings, pastes the key and selects a default team/project
  3. Agent extracts title + description from email, creates issue
- **Proxy needed:** Yes — add `netlify/functions/linear.js`

### Slack
- **What it does:** "Forward to Slack" posts the email content (or a summary) to a Slack channel.
- **API:** Slack Web API (`https://slack.com/api/chat.postMessage`)
- **Auth:** Slack bot token (from a Slack app) or incoming webhook URL
- **Setup steps:**
  1. User creates a Slack app at api.slack.com and gets a bot token
  2. Or simpler: user creates an incoming webhook for a specific channel
  3. In SuperMail settings, pastes the webhook URL or bot token + channel
  4. Agent optionally summarizes the email before posting
- **Proxy needed:** Webhook approach works client-side (no CORS issues). Bot token approach needs a proxy.

---

## Step 5: MCP Integration (Long-Term)

**Goal:** Instead of writing custom integration code for each service, use the Model Context Protocol to standardize how agents expose tools.

MCP is a protocol (developed by Anthropic) where each service runs as an "MCP server" that exposes tools with standardized schemas. The client (SuperMail) connects to MCP servers and the AI model decides which tools to call.

This is the most extensible approach but requires:
- An MCP client in the browser (or a server-side proxy that speaks MCP)
- Each integration packaged as an MCP server (many already exist as open-source packages)
- A way for users to add/configure MCP servers (similar to how Claude Code does it in `claude_desktop_config.json`)

### When to do this
- After 3+ action-oriented agents are working with custom code
- When the pattern of "receive context → call API → return result" is clearly repeating
- When community/third-party integrations become desirable

### How it would work
1. User adds an MCP server config in SuperMail settings (URL + auth)
2. SuperMail discovers available tools from the MCP server
3. When the user forwards to that agent, the AI receives the tools and decides how to use them
4. Results are formatted and saved as an email response

---

## Agent Registry API Reference

Each agent in the registry should implement this interface:

```js
{
  name: 'string',           // Display name (e.g., "Notion")
  icon: 'string',           // Single emoji or character
  match: (to) => boolean,   // Does this To address match this agent?
  needsAuth: boolean,       // Does this agent need user-provided credentials?
  authType: 'apikey' | 'oauth' | 'webhook',
  execute: async (context) => {
    // context = {
    //   thread: [{ from, to, subject, body, date, attachments }],
    //   userMessage: 'string',
    //   email: { subject, threadId, ... },  // the selected email
    //   credentials: { apiKey, token, ... }  // from settings
    // }
    //
    // Returns:
    // {
    //   text: 'string',        // response to show in the email thread
    //   actions: [...]          // optional: side effects performed
    // }
  }
}
```

---

## Quick Reference: Auth & Proxy Requirements

| Agent    | Auth Method        | Proxy Needed? | Google Scope Change? |
|----------|--------------------|---------------|----------------------|
| Gemini   | API key (env var)  | Yes (exists)  | No                   |
| Claude   | API key (env var)  | Yes (new)     | No                   |
| Notion   | Integration token  | Yes (new)     | No                   |
| Calendar | Google OAuth       | No            | Yes (add scope)      |
| Linear   | API key            | Yes (new)     | No                   |
| Slack    | Webhook URL        | No            | No                   |
