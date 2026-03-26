# Web Browse Extended Reference

## Complete Command Reference

### Session Management

| Command | Local | Cloud | Description |
|---------|:-----:|:-----:|-------------|
| `thinkbrowse tabs` | Y | - | List all open Chrome tabs |
| `thinkbrowse attach <tabId>` | Y | - | Attach to a Chrome tab |
| `thinkbrowse session-create` | - | Y | Create a new cloud browser session |
| `thinkbrowse session-status <sid>` | - | Y | Check session status |
| `thinkbrowse session-delete [sid]` | Y | Y | Terminate session |

### Navigation

| Command | Args | Description |
|---------|------|-------------|
| `thinkbrowse navigate <url>` | URL | Navigate (local mode) |
| `thinkbrowse goto <sid> <url>` | session ID, URL | Navigate (cloud mode) |
| `thinkbrowse back [sid]` | session ID | Go back in history |
| `thinkbrowse forward [sid]` | session ID | Go forward in history |
| `thinkbrowse scroll --down <px>` | pixels | Scroll down (local) |
| `thinkbrowse scroll --up <px>` | pixels | Scroll up (local) |
| `thinkbrowse scroll <sid> [up\|down] [pixels]` | session ID | Scroll page (cloud, default: down 500px) |

### Observation

| Command | Args | Description |
|---------|------|-------------|
| `thinkbrowse snapshot [sid]` | session ID | Get accessibility tree (recommended for LLMs) |
| `thinkbrowse screenshot [--output <path>] [sid]` | path, session ID | Capture screenshot |
| `thinkbrowse extract [sid] [text\|html]` | session ID, format | Extract page content |
| `thinkbrowse console [sid]` | session ID | Get browser console logs |
| `thinkbrowse network [sid]` | session ID | Get network requests |
| `thinkbrowse url [sid]` | session ID | Get current page URL |
| `thinkbrowse title [sid]` | session ID | Get current page title |
| `thinkbrowse evaluate [sid] <javascript>` | session ID, JS | Execute JavaScript |

### Interaction

| Command | Args | Description |
|---------|------|-------------|
| `thinkbrowse click [sid] <selector>` | session ID, CSS selector | Click an element |
| `thinkbrowse fill [sid] <selector> <value>` | session ID, CSS selector, value | Clear field and type |
| `thinkbrowse type [sid] <selector> <text>` | session ID, CSS selector, text | Append text |
| `thinkbrowse press [sid] <key>` | session ID, key name | Press key |
| `thinkbrowse hover [sid] <selector>` | session ID, CSS selector | Hover |
| `thinkbrowse select [sid] <selector> <value>` | session ID, CSS selector, option | Select dropdown |
| `thinkbrowse wait [sid] <selector> [timeout_ms]` | session ID, CSS selector | Wait for element (default: 10000ms) |
| `thinkbrowse wait-for-text [sid] <text> [timeout]` | session ID, text | Wait for text (default: 30000ms) |
| `thinkbrowse dialog [sid] [status\|accept\|dismiss]` | session ID, action | Handle dialog |

### Local-Only Commands

| Command | Args | Description |
|---------|------|-------------|
| `thinkbrowse switch-tab <tabId>` | tab ID | Switch Chrome's active tab for visual focus |
| `thinkbrowse clear-logs` | - | Clear console and network logs |

---

## Error Codes

### Cloud Error Codes

| Code | HTTP | Fatal? | Meaning |
|------|------|--------|---------|
| `ELEMENT_NOT_FOUND` | 404 | No | CSS selector matched nothing |
| `NAVIGATION_TIMEOUT` | 504 | No | Page took too long to load |
| `SESSION_CLOSED` | 410 | **Yes** | Browser session closed — create new |
| `SESSION_LIMIT_REACHED` | 429 | No | Max concurrent sessions |
| `PROVISIONING_FAILED` | 503 | No | Cloud machine provisioning failed |
| `INVALID_API_KEY` | 401 | **Yes** | API key invalid |
| `EVALUATE_FAILED` | 400 | **Yes** | JavaScript evaluation error |
| `NETWORK_ERROR` | 502 | No | Target URL unreachable |
| `INTERNAL_ERROR` | 500 | No | Unclassified server error |

### Local/Bridge Error Codes

| Code | Fatal? | Meaning |
|------|--------|---------|
| `EXTENSION_NOT_CONNECTED` | **Yes** | Chrome extension not connected |
| `TAB_NOT_FOUND` | **Yes** | Session tab was closed |
| `TAB_NOT_OWNED` | **Yes** | Agent gated to different tab — re-attach |
| `TAB_OWNED_BY_OTHER_SESSION` | **Yes** | Tab belongs to another session |
| `SESSION_NOT_FOUND` | **Yes** | Session ID not found |
| `SCRIPT_ERROR` | **Yes** | JavaScript evaluation error |

### Retry Decision Tree

```
if (SESSION_CLOSED or TAB_NOT_FOUND or SESSION_NOT_FOUND):
    → STOP. Create a new session.
if (TAB_NOT_OWNED):
    → STOP. Re-attach with thinkbrowse attach <tabId>
if (EVALUATE_FAILED or SCRIPT_ERROR):
    → STOP. Fix the JavaScript.
if (INVALID_API_KEY or EXTENSION_NOT_CONNECTED):
    → STOP. Fix credentials/reconnect.
if (ELEMENT_NOT_FOUND):
    → Take snapshot first, verify selector, retry once.
if (HTTP 502/503/504/524):
    → Automatic retry with exponential backoff.
```

**Circuit breaker**: 2 consecutive fatal errors → abort. 5 transient errors within 60s → abort.

---

## Common Patterns

### Research workflow (cloud)
```bash
SID=$(thinkbrowse session-create)
thinkbrowse goto $SID "https://news.ycombinator.com"
thinkbrowse extract $SID text > /tmp/hn-content.txt
thinkbrowse session-delete $SID
```

### Form submission (cloud)
```bash
SID=$(thinkbrowse session-create)
thinkbrowse goto $SID "https://app.example.com/login"
thinkbrowse fill $SID "#email" "user@example.com"
thinkbrowse fill $SID "#password" "secret"
thinkbrowse click $SID "button[type=submit]"
thinkbrowse wait $SID ".dashboard" 10000
thinkbrowse session-delete $SID
```

### Form submission (local)
```bash
thinkbrowse attach <tabId>
thinkbrowse navigate "https://app.example.com/login"
sleep 2
thinkbrowse type "input[type=email]" "user@example.com"
thinkbrowse type "input[type=password]" "secret"
thinkbrowse click "button[type=submit]"
sleep 3
thinkbrowse screenshot --output "/tmp/logged-in.png"
```

### Dialog handling (cloud)
```bash
SID=$(thinkbrowse session-create)
thinkbrowse goto $SID "https://example.com/settings"
thinkbrowse click $SID "#delete-account"
thinkbrowse dialog $SID status
thinkbrowse dialog $SID accept
thinkbrowse session-delete $SID
```

### Wait for dynamic content (cloud)
```bash
SID=$(thinkbrowse session-create)
thinkbrowse goto $SID "https://example.com/search?q=widgets"
thinkbrowse wait-for-text $SID "results found" 15000
thinkbrowse extract $SID text
thinkbrowse session-delete $SID
```

### React-compatible fill

```bash
# Cloud mode
thinkbrowse evaluate $SID "const el=document.querySelector('#email'); const s=Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set; s.call(el,'user@test.com'); el.dispatchEvent(new Event('input',{bubbles:true}));"
```

---

## REST API Reference

Base URL: `https://api.thinkbrowse.io`
Auth: `X-API-Key` header.

### Session Lifecycle
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/sessions` | Create session |
| GET | `/api/sessions` | List sessions |
| GET | `/api/sessions/:sessionId` | Get session details |
| DELETE | `/api/sessions/:sessionId` | Terminate session |

### Browser Actions
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/sessions/:sid/navigate` | Navigate to URL |
| POST | `/api/sessions/:sid/click` | Click element |
| POST | `/api/sessions/:sid/fill` | Fill form field |
| POST | `/api/sessions/:sid/type` | Type text |
| POST | `/api/sessions/:sid/press` | Press key |
| POST | `/api/sessions/:sid/hover` | Hover element |
| POST | `/api/sessions/:sid/scroll` | Scroll page |
| POST | `/api/sessions/:sid/select` | Select dropdown |
| POST | `/api/sessions/:sid/wait` | Wait for element |
| POST | `/api/sessions/:sid/wait-for-text` | Wait for text |
| POST | `/api/sessions/:sid/dialog` | Handle dialog |
| POST | `/api/sessions/:sid/evaluate` | Execute JavaScript |
| POST | `/api/sessions/:sid/snapshot` | Accessibility tree |
| POST | `/api/sessions/:sid/screenshot` | Take screenshot |
| POST | `/api/sessions/:sid/extract` | Extract page content |

### Session Monitoring
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/sessions/:sid/console` | Console logs |
| GET | `/api/sessions/:sid/network` | Network requests |
| GET | `/api/sessions/:sid/artifacts` | Screenshots (presigned URLs) |

### Task Planning
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tasks/from-text` | Generate task plan from text |
| POST | `/api/tasks/:planId/execute` | Execute task plan |
| GET | `/api/tasks/:planId/status` | Get execution progress |

---

## AI Context Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `thought` | string | Your reasoning (creates audit trail) |
| `agentId` | string | Your agent identifier |
| `captureHtml` | boolean | Capture HTML snapshot after action |
| `captureLogs` | boolean | Capture console logs during action |

---

## Local Mode Extension Troubleshooting

| Directory | Name | All commands? |
|-----------|------|---------------|
| `extension/` | ThinkBrowse Record v0.1.0 | **Yes** — use this one |
| `chrome-extension/` | ThinkBrowse v1.2.0 | No — missing 6 commands |
| `mech-browser-service/chrome-extension/` | Mech Browser Controller v1.0.0 | No — missing wiring |

Fix: Remove old extension, load unpacked from `mech-browse/extension/`.

---

## Architecture

```
Your Brain (thinkbrowse CLI)
    |
    ├── Local mode: Chrome extension + bridge server (localhost:3010)
    |       Chrome (your actual browser)
    |
    └── Cloud mode: X-API-Key → ThinkBrowse API (api.thinkbrowse.io)
                        Fly.io orchestrator
                        Dedicated Fly Machine (per session)
                        Playwright → Chromium (stealth mode)
```

- Session timeout: 30 minutes
- Provisioning time: 60-90 seconds cold start
- Stealth mode: No `navigator.webdriver`
