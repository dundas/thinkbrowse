---
name: web-browse
description: Browse the web programmatically — create cloud browser sessions, navigate pages, interact with elements, extract content
category: browse
sync: all
tags: [browsing, automation, scraping, testing]
---

# Web Browse

Browse the web from any brain using ThinkBrowse. Supports two modes:

- **Local mode**: Controls your actual Chrome browser via the ThinkBrowse extension + bridge server. Best for sites requiring auth, cookies, or WebSocket connections.
- **Cloud mode**: Spins up isolated Playwright browsers on dedicated Fly.io machines. Best for headless/autonomous work, scraping, multi-brain coordination.

`browse.sh` auto-detects which mode to use. Same commands work in both.

## When to Use Which

| Scenario | Mode | Why |
|----------|------|-----|
| Site needs your login cookies | **Local** | Uses your real Chrome session |
| WebSocket-dependent app | **Local** | Cloud can't establish WSS to arbitrary origins |
| Headless autonomous scraping | **Cloud** | No Chrome needed, stealth mode |
| Multi-brain coordination | **Cloud** | Shared infrastructure, API-based |
| Anti-detection needed | **Cloud** | Built-in stealth, no `navigator.webdriver` |
| Interactive development | **Playwright MCP** | Direct DOM access, better DevTools |

## Setup

### Local Mode

1. Install the ThinkBrowse Chrome extension (load unpacked from `mech-browse/extension/`)
   - **Important**: Use the `extension/` directory, NOT `chrome-extension/` or `mech-browser-service/chrome-extension/` — those are older versions missing commands
   - In Chrome: `chrome://extensions` → Developer mode → Load unpacked → select `~/dev_env/mech/mech-browse/extension/`
2. Ensure the native host is running (the extension starts it automatically via Chrome native messaging)
3. Verify: `curl http://127.0.0.1:$(cat ~/.thinkbrowse/port 2>/dev/null || echo 3012)/health` should show `extensionConnected: true`

### Cloud Mode

Set `THINKBROWSE_API_KEY` environment variable:

```bash
export THINKBROWSE_API_KEY="your-api-key"
```

Or create config file at `~/.config/thinkbrowse/config.json`:
```json
{ "apiKey": "your-api-key", "apiUrl": "https://api.thinkbrowse.io" }
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `THINKBROWSE_LOCAL` | - | Set `true` to force local mode |
| `THINKBROWSE_LOCAL_PORT` | - | Native host port override (preferred) |
| `THINKBROWSE_BRIDGE_PORT` | `3012` | Legacy port override (use `THINKBROWSE_LOCAL_PORT` instead) |
| `THINKBROWSE_API_KEY` | - | Cloud API key |
| `THINKBROWSE_URL` | `https://api.thinkbrowse.io` | Cloud API base URL |

Config file: `~/.config/thinkbrowse/config.json` with `apiKey` and `apiUrl` fields.

### Mode Auto-Detection

`browse.sh` checks in order:
1. `THINKBROWSE_LOCAL=true` env var → **local mode**
2. Native host running at `127.0.0.1:<port>` (port auto-discovered from `~/.thinkbrowse/port`, default 3012) with extension connected → **local mode**
3. `THINKBROWSE_API_KEY` env var or config file → **cloud mode**
4. Neither → error with setup instructions

### Local Mode Window Isolation

Each `session-create` opens a **new Chrome window** (unfocused, starting at `about:blank`). This gives every agent its own isolated window so that:

- **Screenshots capture the correct tab by default** — the agent's tab is the active tab in its own window
- **Multiple agents run simultaneously** without normally blocking each other
- **User browsing is uninterrupted** — agent windows open in the background
- **No tab conflicts** — each session gets its own window

All subsequent commands for that session are pinned to the agent's tab via `X-Tab-Id` header.

After `thinkbrowse attach <tabId>`, the agent is **gated to that tab** — all commands route exclusively to the attached tab via the `X-Tab-Id` header. Sending commands on unowned tabs returns `TAB_NOT_OWNED` (fatal — re-attach required).

> **Prefer `session-create` over `attach` for agent work.** `attach` places the agent on a tab in the user's existing window, which means no isolation — screenshots require focus-switching, and user activity can interfere. Use `attach` only when you need to interact with a specific existing tab (e.g. one that's already logged in).

- Session-to-tab mappings are stored in `/tmp/thinkbrowse-tabs/<sessionId>`
- `session-delete` closes the tab (and its window if empty) and cleans up
- If the user manually closes the tab, commands return `TAB_NOT_FOUND` and auto-clean up
- Multiple sessions = multiple windows, each independent
- **Always clean up**: run `session-delete` when done to avoid window accumulation

#### Multi-Agent Usage

Multiple agents can browse simultaneously on the same Chrome instance:

```bash
# Agent 1 (runs in its own background window)
SID1=$(bash $BROWSE session-create)
bash $BROWSE goto $SID1 "https://site-a.com"

# Agent 2 (runs in a separate background window)
SID2=$(bash $BROWSE session-create)
bash $BROWSE goto $SID2 "https://site-b.com"

# Both can screenshot, navigate, click, etc. without interfering
bash $BROWSE screenshot $SID1   # captures Agent 1's window
bash $BROWSE screenshot $SID2   # captures Agent 2's window

# Clean up both
bash $BROWSE session-delete $SID1
bash $BROWSE session-delete $SID2
```

The user can browse normally in their own Chrome windows throughout — agent windows open unfocused and should not steal focus.

## Quick Start (Bash)

```bash
BROWSE="scripts/browse.sh"

# Create a session (returns session ID when ready)
# Local mode: opens a new unfocused Chrome window (isolated from your browsing)
# Cloud mode: provisions a dedicated Playwright browser on Fly.io (~60-90s)
SID=$(bash $BROWSE session-create)

# Navigate
bash $BROWSE goto $SID "https://example.com"

# Get accessibility tree (best for LLM understanding)
bash $BROWSE snapshot $SID

# Extract text content
bash $BROWSE extract $SID text

# Interact with elements
bash $BROWSE click $SID "button.submit"
bash $BROWSE fill $SID "#email" "user@example.com"
bash $BROWSE type $SID "#search" "query text"
bash $BROWSE press $SID Enter

# Browser history
bash $BROWSE back $SID
bash $BROWSE forward $SID

# Screenshot (base64 image)
bash $BROWSE screenshot $SID

# Page info
bash $BROWSE url $SID
bash $BROWSE title $SID

# Observability
bash $BROWSE console $SID
bash $BROWSE network $SID

# Clean up (always do this!)
bash $BROWSE session-delete $SID
```

### Local Mode Troubleshooting

If commands like `fill`, `press`, `hover`, `select`, `wait`, or `scroll` return "Unknown command", you have the **wrong extension installed**. The repo has 3 extension directories:

| Directory | Name | All commands? |
|-----------|------|---------------|
| `extension/` | ThinkBrowse Record v0.1.0 | **Yes** — use this one |
| `chrome-extension/` | ThinkBrowse v1.2.0 | No — missing 6 commands |
| `mech-browser-service/chrome-extension/` | Mech Browser Controller v1.0.0 | No — missing wiring |

Fix: In Chrome `chrome://extensions`, remove the old extension and load unpacked from `mech-browse/extension/`.

**React-compatible fill**: The `fill` command uses native DOM value setting. If a React controlled input still doesn't update state, use the evaluate workaround:
```bash
bash $BROWSE evaluate $SID "const el=document.querySelector('#email'); const s=Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set; s.call(el,'user@test.com'); el.dispatchEvent(new Event('input',{bubbles:true}));"
```

## Quick Start (TypeScript/Bun)

> Note: The TypeScript client supports cloud mode only. For local mode, use `browse.sh` (bash).

```typescript
import { ThinkBrowseClient } from './.claude/skills/web-browse/browse-client';

const browser = new ThinkBrowseClient();
const sid = await browser.createSession();

try {
  await browser.navigate(sid, 'https://example.com');
  const snapshot = await browser.snapshot(sid);
  console.log(snapshot);

  await browser.click(sid, 'a.login');
  await browser.fill(sid, '#email', 'user@test.com');
  await browser.fill(sid, '#password', 'secret');
  await browser.click(sid, 'button[type=submit]');

  await browser.wait(sid, '.dashboard', 10000);
  const content = await browser.extract(sid, 'text');
  console.log(content);
} finally {
  await browser.deleteSession(sid);
}
```

**TypeScript client methods**: `createSession`, `sessionStatus`, `deleteSession`, `navigate`, `scroll`, `snapshot`, `screenshot`, `extract`, `click`, `fill`, `type`, `press`, `hover`, `select`, `wait`, `evaluate`. For `back`, `forward`, `dialog`, `waitForText`, `console`, `network`, `url`, `title`, `html` — use `browse.sh` or call the REST API directly.

## Complete Command Reference

### Session Management

| Command | Args | Description |
|---------|------|-------------|
| `session-create` | - | Create a new browser session, returns session ID when ready |
| `session-status <sid>` | session ID | Check session status and details |
| `session-delete <sid>` | session ID | Terminate session and free resources |

**Cloud session status progression**: `creating` → `pending` → `provisioning` → `ready`/`running`

**Cloud session polling**: 40 attempts at 3s intervals = 120s max wait.

### Navigation

| Command | Args | Description |
|---------|------|-------------|
| `goto <sid> <url>` | session ID, URL | Navigate to URL |
| `back <sid>` | session ID | Go back in browser history |
| `forward <sid>` | session ID | Go forward in browser history |
| `scroll <sid> [up\|down] [pixels]` | session ID, direction, amount | Scroll page (default: down 500px) |

### Observation

| Command | Args | Description |
|---------|------|-------------|
| `snapshot <sid>` | session ID | Get accessibility tree (recommended for LLMs) |
| `screenshot <sid>` | session ID | Capture screenshot as base64 |
| `extract <sid> [text\|html]` | session ID, format | Extract page content (default: text) |
| `console <sid>` | session ID | Get browser console logs |
| `network <sid>` | session ID | Get network requests |
| `url <sid>` | session ID | Get current page URL |
| `title <sid>` | session ID | Get current page title |
| `html <sid>` | session ID | Get full page HTML |

### Interaction

| Command | Args | Description |
|---------|------|-------------|
| `click <sid> <selector>` | session ID, CSS selector | Click an element |
| `fill <sid> <selector> <value>` | session ID, CSS selector, value | Clear field and type value |
| `type <sid> <selector> <text>` | session ID, CSS selector, text | Append text to field |
| `press <sid> <key>` | session ID, key name | Press key (Enter, Tab, Escape, ArrowDown, etc.) |
| `hover <sid> <selector>` | session ID, CSS selector | Hover over element |
| `select <sid> <selector> <value>` | session ID, CSS selector, option value | Select dropdown option |
| `wait <sid> <selector> [timeout_ms]` | session ID, CSS selector, timeout | Wait for element to appear (default: 10000ms) |
| `wait-for-text <sid> <text> [timeout]` | session ID, text, timeout | Wait for text to appear on page (default: 30000ms) |
| `dialog <sid> [status\|accept\|dismiss]` | session ID, action | Handle browser dialog (alert/confirm/prompt) |
| `evaluate <sid> <javascript>` | session ID, JS expression | Execute JavaScript on page |

### Local-Only Commands

| Command | Args | Description |
|---------|------|-------------|
| `tabs` | - | List all open Chrome tabs |
| `switch-tab <sid> <tabId>` | session ID, tab ID | Switch to a specific tab |
| `clear-logs <sid>` | session ID | Clear console and network logs |

### Mode Support Matrix

| Command | Local | Cloud |
|---------|:-----:|:-----:|
| session-create / status / delete | Y | Y |
| goto / back / forward / scroll | Y | Y |
| snapshot / screenshot / extract | Y | Y |
| click / fill / type / press / hover / select | Y | Y |
| wait / wait-for-text / dialog / evaluate | Y | Y |
| console / network / url / title / html | Y | Y |
| tabs / switch-tab / clear-logs | Y | - |

## Common Patterns

### Research workflow
```bash
BROWSE="scripts/browse.sh"
SID=$(bash $BROWSE session-create)
bash $BROWSE goto $SID "https://news.ycombinator.com"
bash $BROWSE extract $SID text > /tmp/hn-content.txt
bash $BROWSE session-delete $SID
```

### Form submission
```bash
SID=$(bash $BROWSE session-create)
bash $BROWSE goto $SID "https://app.example.com/login"
bash $BROWSE fill $SID "#email" "user@example.com"
bash $BROWSE fill $SID "#password" "secret"
bash $BROWSE click $SID "button[type=submit]"
bash $BROWSE wait $SID ".dashboard" 10000
bash $BROWSE screenshot $SID > /tmp/logged-in.json
bash $BROWSE session-delete $SID
```

### Multi-page navigation with history
```bash
SID=$(bash $BROWSE session-create)
bash $BROWSE goto $SID "https://example.com"
bash $BROWSE click $SID "a.products"
bash $BROWSE click $SID ".product-card:first-child"
bash $BROWSE back $SID          # back to product list
bash $BROWSE back $SID          # back to home
bash $BROWSE forward $SID       # forward to product list
bash $BROWSE session-delete $SID
```

### Dialog handling
```bash
SID=$(bash $BROWSE session-create)
bash $BROWSE goto $SID "https://example.com/settings"
bash $BROWSE click $SID "#delete-account"
# Check if a confirmation dialog appeared
bash $BROWSE dialog $SID status
# Accept or dismiss
bash $BROWSE dialog $SID accept
bash $BROWSE session-delete $SID
```

### Wait for dynamic content
```bash
SID=$(bash $BROWSE session-create)
bash $BROWSE goto $SID "https://example.com/search?q=widgets"
bash $BROWSE wait-for-text $SID "results found" 15000
bash $BROWSE extract $SID text
bash $BROWSE session-delete $SID
```

## Error Codes

All errors return structured JSON with a machine-readable `code` and optional `hint`:

```json
{
  "success": false,
  "error": "Element not found: button.submit",
  "code": "ELEMENT_NOT_FOUND",
  "hint": "Verify the selector exists on the current page. Try taking a snapshot first.",
  "retryable": true
}
```

### Cloud Error Codes

| Code | HTTP | Fatal? | Meaning |
|------|------|--------|---------|
| `ELEMENT_NOT_FOUND` | 404 | No | CSS selector matched nothing |
| `NAVIGATION_TIMEOUT` | 504 | No | Page took too long to load |
| `OPERATION_TIMEOUT` | 504 | No | Operation exceeded time limit |
| `SESSION_CLOSED` | 410 | **Yes** | Browser session was closed — create a new one |
| `PAGE_NAVIGATED` | 409 | No | Navigation occurred during operation |
| `BROWSER_PROTOCOL_ERROR` | 502 | No | Browser internal error |
| `STORAGE_TIMEOUT` | 503 | No | Storage backend slow |
| `STORAGE_ERROR` | 502 | No | Storage backend error |
| `BROWSER_POOL_UNAVAILABLE` | 503 | No | No browser instances available |
| `SESSION_LIMIT_REACHED` | 429 | No | Max concurrent sessions — delete unused sessions |
| `PROVISIONING_FAILED` | 503 | No | Cloud machine provisioning failed (retries exhausted) |
| `INVALID_API_KEY` | 401 | **Yes** | API key invalid or expired |
| `EVALUATE_FAILED` | 400 | **Yes** | JavaScript evaluation error — fix the expression |
| `PAYLOAD_TOO_LARGE` | 413 | No | Request body exceeds size limit |
| `NETWORK_ERROR` | 502 | No | Target URL unreachable |
| `INTERNAL_ERROR` | 500 | No | Unclassified server error |

### Local/Bridge Error Codes

| Code | Fatal? | Meaning |
|------|--------|---------|
| `EXTENSION_NOT_CONNECTED` | **Yes** | Chrome extension not connected to bridge |
| `EXTENSION_DISCONNECTED` | **Yes** | Extension disconnected mid-operation |
| `TAB_NOT_FOUND` | **Yes** | Session tab was closed — run `thinkbrowse attach <tabId>` or create new session |
| `TAB_NOT_OWNED` | **Yes** | Agent is gated to a different tab after attach — re-attach to target tab |
| `TAB_OWNED_BY_OTHER_SESSION` | **Yes** | Tab belongs to another session — cannot operate on it |
| `SESSION_NOT_FOUND` | **Yes** | Session ID not found in tab registry — create a new session |
| `REQUEST_TIMEOUT` | No | Bridge request timed out |
| `COMMAND_TIMEOUT` | No | Command execution timed out |
| `ELEMENT_NOT_FOUND` | No | Selector matched nothing |
| `NAVIGATION_FAILED` | No | Navigation error |
| `SCRIPT_ERROR` | **Yes** | JavaScript evaluation error |

## Retry Behavior

### Cloud mode (browse.sh)
- **Max retries**: 4 attempts
- **Backoff**: exponential — 1s, 2s, 4s, 8s
- **Retryable conditions**: HTTP 502/503/504/524, `retryable: true` in response, "Session not found", "Session is still provisioning"

### Local mode (browse.sh)
- **Max retries**: 3 attempts
- **Backoff**: exponential — 1s, 2s, 4s
- **Retryable conditions**: HTTP 503/504, `retryable: true` in response
- **Auto-cleanup**: `TAB_NOT_FOUND` removes session immediately (no retry)

### TypeScript client (browse-client.ts)
- **Max retries**: 4 attempts (configurable via `maxRetries` constructor option)
- **Backoff**: linear — 1s, 2s, 3s, 4s
- **Retryable conditions**: HTTP 504, "Session not found", "Session is still provisioning"

### Retry Decision Tree

```
if (code is SESSION_CLOSED or TAB_NOT_FOUND or SESSION_NOT_FOUND):
    → STOP. Create a new session.
if (code is TAB_NOT_OWNED):
    → STOP. Re-attach: run `thinkbrowse attach <tabId>` for the target tab.
if (code is TAB_OWNED_BY_OTHER_SESSION):
    → STOP. Cannot access this tab — switch to a session that owns it.
if (code is EVALUATE_FAILED or SCRIPT_ERROR):
    → STOP. Fix the JavaScript expression.
if (code is INVALID_API_KEY or EXTENSION_NOT_CONNECTED):
    → STOP. Fix credentials or reconnect extension.
if (code is ELEMENT_NOT_FOUND):
    → Take a snapshot first. Verify selector. Retry once.
if (HTTP is 502/503/504/524):
    → Automatic retry with exponential backoff.
if (code is SESSION_LIMIT_REACHED):
    → Delete unused sessions, then retry.
else:
    → Check the `hint` field for recovery advice.
```

**Circuit breaker** (for automated task loops):
- 2 consecutive fatal errors → abort immediately
- 5 transient errors within 60 seconds → abort
- Fatal codes: `SESSION_CLOSED`, `TAB_NOT_FOUND`, `TAB_NOT_OWNED`, `TAB_OWNED_BY_OTHER_SESSION`, `SESSION_NOT_FOUND`, `SCRIPT_ERROR`, `EVALUATE_FAILED`, `EXTENSION_NOT_CONNECTED`

## AI Context Parameters

Most action endpoints accept optional parameters for audit trails and evidence capture:

| Parameter | Type | Description |
|-----------|------|-------------|
| `thought` | string | Your reasoning for this action (creates audit trail) |
| `agentId` | string | Your agent identifier |
| `captureHtml` | boolean | Capture page HTML snapshot after action |
| `captureLogs` | boolean | Capture console logs during action |
| `multiple` | boolean | On `extract`: return array of all matching elements |

Example — click with evidence capture:
```bash
curl -s -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -X POST "https://api.thinkbrowse.io/api/sessions/$SID/click" \
  -d '{"selector":"#confirm","thought":"Confirming purchase order","captureHtml":true}'
```

## REST API Reference

Base URL: `https://api.thinkbrowse.io` (or `https://mech-browser-service.fly.dev`)
Auth: `X-API-Key` header on all endpoints except health check.

### Session Lifecycle

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/sessions` | Create session (returns sessionId) |
| GET | `/api/sessions` | List all sessions for user |
| GET | `/api/sessions/:sessionId` | Get session details (storage-first) |
| DELETE | `/api/sessions/:sessionId` | Terminate session |
| GET | `/api/sessions/history` | Historical sessions with pagination |

### Browser Actions

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/sessions/:sid/navigate` | Navigate to URL |
| POST | `/api/sessions/:sid/go-back` | Go back in history |
| POST | `/api/sessions/:sid/go-forward` | Go forward in history |
| POST | `/api/sessions/:sid/click` | Click element |
| POST | `/api/sessions/:sid/fill` | Fill form field |
| POST | `/api/sessions/:sid/type` | Type text |
| POST | `/api/sessions/:sid/press` | Press key |
| POST | `/api/sessions/:sid/hover` | Hover element |
| POST | `/api/sessions/:sid/scroll` | Scroll page |
| POST | `/api/sessions/:sid/select` | Select dropdown option |
| POST | `/api/sessions/:sid/wait` | Wait for element |
| POST | `/api/sessions/:sid/wait-for-text` | Wait for text |
| POST | `/api/sessions/:sid/dialog` | Handle dialog (accept/dismiss) |
| GET | `/api/sessions/:sid/dialog` | Get dialog state |
| POST | `/api/sessions/:sid/evaluate` | Execute JavaScript |
| POST | `/api/sessions/:sid/snapshot` | Accessibility tree |
| POST | `/api/sessions/:sid/screenshot` | Take screenshot |
| POST | `/api/sessions/:sid/extract` | Extract page content |

### Session Monitoring

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/sessions/:sid/actions` | Action history with filtering |
| GET | `/api/sessions/:sid/health` | Queue health metrics |
| GET | `/api/sessions/:sid/details` | Full session details from storage |
| GET | `/api/sessions/:sid/artifacts` | Screenshots and recordings (presigned URLs) |
| GET | `/api/sessions/:sid/console` | Console logs |
| GET | `/api/sessions/:sid/network` | Network requests |

### Task Planning & Execution

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tasks/from-text` | Generate task plan from text instruction |
| POST | `/api/tasks/from-audio` | Generate task plan from audio (Whisper STT) |
| POST | `/api/tasks/from-recording` | Generate task plan from video recording |
| GET | `/api/tasks` | List all task plans for user |
| GET | `/api/tasks/:planId` | Get task plan details |
| GET | `/api/tasks/by-session/:sid` | Get plans for a session |
| POST | `/api/tasks/:planId/execute` | Execute task plan |
| POST | `/api/tasks/:planId/pause` | Pause execution |
| POST | `/api/tasks/:planId/resume` | Resume execution |
| POST | `/api/tasks/:planId/cancel` | Cancel execution |
| PATCH | `/api/tasks/:planId/tasks/:taskId` | Update individual task status |
| GET | `/api/tasks/:planId/status` | Get execution progress |
| DELETE | `/api/tasks/:planId` | Delete task plan |

### CLI Agent Execution

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/agent/execute` | Execute browser instruction via AI agent |
| GET | `/api/agent/:taskId/status` | Get agent task status |
| POST | `/api/agent/:taskId/cancel` | Cancel agent task |

### Analytics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/analytics/overview` | Usage overview (preset/date range) |
| GET | `/api/analytics/failures` | Failure breakdown and top errors |
| GET | `/api/analytics/costs` | Cost analytics |
| GET | `/api/analytics/trends` | Trends over time |

### API Keys

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/auth/api-keys` | List active API keys |
| POST | `/api/auth/api-keys` | Create new API key |
| DELETE | `/api/auth/api-keys/:keyId` | Revoke API key |

### Feedback Recordings

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/feedback/recordings` | Upload human recording |
| GET | `/api/feedback/recordings` | List recordings |
| GET | `/api/feedback/recordings/:id` | Get recording metadata |
| GET | `/api/feedback/recordings/:id/video` | Download recording video |
| POST | `/api/feedback/recordings/:id/analyze` | Trigger recording analysis |
| DELETE | `/api/feedback/recordings/:id` | Delete recording |

### Browser Profiles

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/profiles` | Create browsing profile |
| GET | `/api/profiles` | List profiles |
| GET | `/api/profiles/:id` | Get profile |
| PUT | `/api/profiles/:id` | Update profile |
| DELETE | `/api/profiles/:id` | Delete profile |

## Task Planning API

For autonomous multi-step workflows, use the task planner instead of manual step-by-step commands:

```bash
BASE="https://api.thinkbrowse.io"
KEY="your-api-key"

# Create a plan from natural language
PLAN=$(curl -s -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -X POST "$BASE/api/tasks/from-text" \
  -d '{"task":"Login to example.com and extract the user profile"}' \
  | jq -r '.planId')

# Execute the plan (creates a session automatically)
curl -s -H "X-API-Key: $KEY" -X POST "$BASE/api/tasks/$PLAN/execute"

# Check status
curl -s -H "X-API-Key: $KEY" "$BASE/api/tasks/$PLAN/status"

# Get full plan with task details
curl -s -H "X-API-Key: $KEY" "$BASE/api/tasks/$PLAN"
```

## Live Streaming

Watch the browser in real-time via WebSocket:
```
wss://api.thinkbrowse.io/stream/{sessionId}
```

## Architecture

```
Your Brain (bash/bun)
    | curl + X-API-Key
ThinkBrowse API (api.thinkbrowse.io)
    | Fly.io orchestrator
Dedicated Fly Machine (per session)
    | Playwright
Chromium (stealth mode)
    |
Target Website
```

Each cloud session gets its own dedicated Fly.io machine with Playwright. The orchestrator routes requests to the correct worker machine via `fly-replay` headers. Sessions are stored in mech-storage (NoSQL) as the single source of truth — no in-memory state on orchestrators.

- **Session timeout**: 30 minutes (auto-cleanup)
- **Provisioning time**: 60-90 seconds for cold start
- **Machine retry**: If first machine fails, automatically retries on a second machine
- **Stealth mode**: Built-in anti-detection (no `navigator.webdriver`)

## Reference Links

- **Interactive API docs**: `GET /api-docs/` (Swagger)
- **OpenAPI spec**: `GET /openapi.json`
- **Health check**: `GET /health`
- **Agent guide**: `GET /llms.txt`
- **Website docs**: `https://thinkbrowse.io/docs`
