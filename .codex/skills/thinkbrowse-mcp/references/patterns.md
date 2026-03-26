# ThinkBrowse MCP — All Patterns & Gotchas

## Reliable Patterns

### 1. Create → Wait → Navigate → Observe
```
session_create → session_wait_ready → navigate → snapshot → screenshot
```

### 2. Find Elements Before Clicking
```
snapshot  {thought: "Find selector"}
click  {selector: "button[type=submit]", thought: "Confirmed via snapshot"}
```

### 3. Click by Text (Ambiguous Selectors)
```
evaluate  {script: "[...document.querySelectorAll('button,a')].find(el=>el.textContent.trim()==='Sign In')?.click()"}
```

### 4. React Forms (type_text, NOT fill)
```
click → type_text → click → type_text → click submit
```

### 5. Scroll + Screenshot Loop
```
screenshot → scroll down → screenshot → scroll down → screenshot
```

### 6. List Interactive Elements
```
evaluate  {script: "[...document.querySelectorAll('button,[role=tab],a')].map(b=>({label:b.getAttribute('aria-label'),text:b.textContent.trim().slice(0,50)})).filter(b=>b.text||b.label)"}
```

### 7. Check Element Existence (No Error)
```
evaluate  {script: "document.querySelector('.target')?.textContent?.trim()"}
```
Returns `null` if not found.

### 8. Wait for Dynamic Content
```
navigate → wait_for_text "results found" → extract
```

### 9. Mode Switching
```
set_mode {mode: "local"} → session_create → navigate (with cookies)
```

### 10. Session Cleanup
```
session_delete  {sessionId: "...", thought: "Done — cleaning up"}
```

### 11. Selector Scroll (Into View)
```
evaluate  {script: "document.querySelector('.element')?.scrollIntoView({behavior:'smooth',block:'center'})"}
```

### 12. Extract Visible Text (Safe for Large Pages)
```
evaluate  {script: "([...document.querySelectorAll('h1,h2,h3,p,span')].map(e=>e.textContent.trim()).filter(t=>t.length>4).join('\\n')).slice(0,2000)"}
```

### 13. Shadow DOM Traversal (Reddit, YouTube, Modern SPAs)

Standard `click`/`extract` can't reach elements inside Shadow DOM. Use `evaluate` with recursive traversal:

**Click by text inside Shadow DOM:**
```
evaluate  {
  script: "(function(){function f(r){var els=r.querySelectorAll('*');for(var i=0;i<els.length;i++){var el=els[i];if(el.shadowRoot){var res=f(el.shadowRoot);if(res)return res}var t=(el.textContent||'').trim();if(t==='TARGET_TEXT'&&el.children.length===0){var c=el;var p=el.parentElement;while(p){if(p.tagName==='BUTTON'||p.tagName==='A'||p.getAttribute('role')==='button'){c=p;break}p=p.parentElement}c.click();return 'Clicked: '+c.tagName}}return null}return f(document)||'not found'})()",
  thought: "Click element inside Shadow DOM by visible text"
}
```

**List interactive elements inside Shadow DOM:**
```
evaluate  {
  script: "(function(){function f(r,res){var els=r.querySelectorAll('button,a,[role=button]');for(var i=0;i<els.length;i++){var t=(els[i].textContent||'').trim();if(t.length>0&&t.length<50)res.push(t)}var all=r.querySelectorAll('*');for(var i=0;i<all.length;i++){if(all[i].shadowRoot)f(all[i].shadowRoot,res)}return res}return f(document,[]).slice(0,20)})()",
  thought: "List all clickable elements including those inside Shadow DOM"
}
```

**Works for:** reading content, clicking buttons/links, navigating, extracting text.
**Fails for:** Reddit comment posting, some anti-bot form submissions — require genuine browser-level user gestures. Use handoff for these cases.

## Anti-Patterns

| # | Anti-Pattern | Fix |
|---|-------------|-----|
| 1 | Clicking without observing | `snapshot` first, then click with specific selector |
| 2 | `fill` on React inputs | Use `type_text` — keyboard events trigger onChange |
| 3 | Skip `session_wait_ready` | Cloud provisioning takes 60-90s — always wait |
| 4 | Missing `thought` parameter | Always include reasoning for audit trail |
| 5 | Async `evaluate` expecting return | Promises return empty. Use `network_requests` |
| 6 | `document.body.innerText` | Target specific selectors, `.slice(0,2000)` |
| 7 | Not deleting sessions | Always `session_delete` when done |

## Gotchas Reference

| Gotcha | Details |
|--------|---------|
| `fill` vs `type_text` | `fill` = native DOM. `type_text` = keyboard events (React). |
| `evaluate` async | Returns empty for Promises. Synchronous only. |
| Cloud provisioning | 60-90s. Always `session_wait_ready`. |
| Click by text | CSS selectors only. Use `evaluate` + `querySelectorAll().find()`. |
| Click ≠ navigation | Click returns immediately. `wait_for_element` after links. |
| `no_op_click` | Use `evaluate` to click programmatically. |
| Protected flows | Stripe/OAuth: `blocked: true, handoffRequired: true`. Stop. |
| Stale tab locks | Crashed agent leaves lock. Use different tab. |
| evaluate auto-wrap | Single expressions OK. Multi-statement: `(()=>{...})()` |
| CSRF forms | Navigate → wait 2s → fill → submit. |
| Session timeout | Cloud: 30min auto-cleanup. Always clean up. |
| Screenshot format | MCP returns base64 data. CLI returns file path. |
