# ThinkBrowse CLI — Error Codes & Recovery Playbooks

## Fatal Errors (Stop Immediately)

| Code | HTTP | Action |
|------|------|--------|
| `SESSION_CLOSED` | 410 | Create new session — this one is gone |
| `TAB_NOT_FOUND` | 410 | `thinkbrowse tabs` → `thinkbrowse attach <newTabId>` |
| `TAB_NOT_OWNED` | 403 | Re-attach: `thinkbrowse attach <tabId>` |
| `TAB_OWNED_BY_OTHER_SESSION` | 403 | Use a different tab — another agent owns this one |
| `SESSION_NOT_FOUND` | 404 | Create new session |
| `INVALID_API_KEY` | 401 | `thinkbrowse config set apiKey <key>` |
| `INSUFFICIENT_CREDITS` | 402 | Purchase credits |
| `EVALUATE_FAILED` | 400 | Fix JavaScript syntax |
| `SCRIPT_ERROR` | 400 | Fix JavaScript expression |
| `NATIVE_HOST_ERROR` | - | `thinkbrowse doctor` — bridge not running |

## Retryable Errors (Backoff: 1s/2s/4s, max 3)

| Code | HTTP | Recovery |
|------|------|----------|
| `ELEMENT_NOT_FOUND` | 404 | Take snapshot, verify selector, retry once |
| `ELEMENT_IN_IFRAME` | 404 | Use frame-aware selector |
| `ELEMENT_IN_SHADOW_DOM` | 404 | Use `evaluate` with recursive Shadow DOM traversal — see Pattern 13 in patterns.md |
| `NAVIGATION_TIMEOUT` | 504 | Increase timeout or check URL |
| `OPERATION_TIMEOUT` | 504 | Simplify operation |
| `PAGE_NAVIGATED` | 409 | Wait for navigation to complete, retry |
| `BROWSER_PROTOCOL_ERROR` | 502 | Retry; create new session if persists |
| `NETWORK_ERROR` | 502 | Check URL accessibility |
| `INTERNAL_ERROR` | 500 | Retry once |

## Special Cases

### SESSION_LIMIT_REACHED (429)
Do NOT retry blindly. Clean up first:
```bash
thinkbrowse cloud list                    # Find sessions to delete
thinkbrowse cloud stop -s <old-session>   # Free a slot
thinkbrowse cloud start "https://..."     # Now create
```

### ELEMENT_NOT_FOUND — Recovery Playbook
```bash
# 1. Inspect page structure
thinkbrowse snapshot

# 2. List all interactive elements
thinkbrowse evaluate "[...document.querySelectorAll('button,a,input,select')].map(e=>({tag:e.tagName,id:e.id,class:e.className.slice(0,30),text:e.textContent.trim().slice(0,50)}))"

# 3. Click by text instead
thinkbrowse evaluate "[...document.querySelectorAll('button')].find(b=>b.textContent.includes('Submit'))?.click()"
```

### no_op_click (Click Blocked / No State Change)
Element exists but click has no effect. Common with tab-like elements.
```bash
# Use evaluate to click programmatically
thinkbrowse evaluate "document.querySelector('[data-tab=recordings]')?.click()"

# Or find by text
thinkbrowse evaluate "[...document.querySelectorAll('[role=tab]')].find(t=>t.textContent.includes('Recordings'))?.click()"
```

### Tab Unresponsive / Fetch Failed
```bash
thinkbrowse tabs                          # Find responsive tab
thinkbrowse attach <workingTabId>
sleep 2
thinkbrowse url                           # Verify connection
```

### Protected Flows (Stripe, OAuth)
Cross-origin iframes return:
```json
{"success": false, "blocked": true, "handoffRequired": true, "category": "secure_hosted_field"}
```
Stop automation, notify user, wait for manual completion.

### Circuit Breaker (Automated Loops)
- 2 consecutive fatal errors → abort immediately
- 5 transient errors within 60 seconds → abort
