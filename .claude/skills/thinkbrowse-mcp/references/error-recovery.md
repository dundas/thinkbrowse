# ThinkBrowse MCP — Error Codes & Recovery Playbooks

## Fatal Errors (Stop Immediately)

| Code | HTTP | Action |
|------|------|--------|
| `SESSION_CLOSED` | 410 | `session_create` → `session_wait_ready` → resume |
| `TAB_NOT_FOUND` | 410 | `tab_list` → `session_create` with new tab |
| `TAB_NOT_OWNED` | 403 | Use a different tab — another agent owns this one |
| `SESSION_NOT_FOUND` | 404 | Create new session |
| `INVALID_API_KEY` | 401 | Check THINKBROWSE_API_KEY env var |
| `INSUFFICIENT_CREDITS` | 402 | Purchase credits |
| `EVALUATE_FAILED` | 400 | Fix JavaScript syntax in script param |

## Retryable Errors (Backoff: 1s/2s/4s, max 3)

| Code | HTTP | Recovery |
|------|------|----------|
| `ELEMENT_NOT_FOUND` | 404 | `snapshot` → verify selector → retry |
| `NAVIGATION_TIMEOUT` | 504 | Retry or increase timeout |
| `OPERATION_TIMEOUT` | 504 | Simplify the operation |
| `PAGE_NAVIGATED` | 409 | `wait_for_element` → retry |
| `BROWSER_PROTOCOL_ERROR` | 502 | Retry; new session if persists |
| `NETWORK_ERROR` | 502 | Check URL accessibility |
| `INTERNAL_ERROR` | 500 | Retry once |

## Special Cases

### SESSION_LIMIT_REACHED (429)
Do NOT retry. Clean up first:
```
1. session_list  {thought: "Find sessions to delete"}
2. session_delete  {sessionId: "old", thought: "Free slot"}
3. session_create  {thought: "Now create needed session"}
```

### ELEMENT_NOT_FOUND — Recovery Playbook
```
1. snapshot  {thought: "Inspect page structure"}
2. evaluate  {
     script: "[...document.querySelectorAll('button,a,input')].map(e=>({tag:e.tagName,id:e.id,text:e.textContent.trim().slice(0,40)}))",
     thought: "List interactive elements"
   }
3. click  {selector: "<corrected>", thought: "Retry with correct selector"}
```

### no_op_click
Element exists but click has no effect:
```
evaluate  {
  script: "document.querySelector('[data-tab=recordings]')?.click()",
  thought: "Click programmatically — direct click had no effect"
}
```

### Protected Flows (Stripe, OAuth)
Returns `{"blocked": true, "handoffRequired": true}`. Stop automation, notify user.

### Circuit Breaker
- 2 consecutive fatal errors → abort
- 5 transient errors within 60s → abort
