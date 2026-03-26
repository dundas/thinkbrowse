# ThinkBrowse CLI — All Patterns & Gotchas

## Reliable Patterns

### 1. Navigate and Observe
```bash
thinkbrowse navigate "https://example.com"
sleep 2
thinkbrowse snapshot
thinkbrowse screenshot --output /tmp/page.png
```

### 2. Click by Text Content (RECOMMENDED)
```bash
thinkbrowse evaluate "[...document.querySelectorAll('button,a,[role=button]')].find(el=>el.textContent.trim()==='Sign In')?.click()"
sleep 1
```

### 3. Extract All Text Content
```bash
thinkbrowse evaluate "([...document.querySelectorAll('h1,h2,h3,p,label,span')].map(e=>e.textContent.trim()).filter(t=>t.length>4).join('\n')).slice(0,2000)"
```

### 4. List All Interactive Elements
```bash
thinkbrowse evaluate "[...document.querySelectorAll('button,[role=tab],[role=button]')].map(b=>({label:b.getAttribute('aria-label'),text:b.textContent.trim(),class:b.className.slice(0,50)})).filter(b=>b.text||b.label)"
```

### 5. Fill React Forms
```bash
thinkbrowse click "input[type=email]"
thinkbrowse type "input[type=email]" "user@example.com"
thinkbrowse click "input[type=password]"
thinkbrowse type "input[type=password]" "password123"
thinkbrowse click "button[type=submit]"
sleep 3
```

### 6. Scroll and Capture
```bash
thinkbrowse scroll --direction down --amount 600
sleep 1
thinkbrowse screenshot --output /tmp/scrolled.png
```

### 7. Check Element Existence (No Error)
```bash
thinkbrowse evaluate "document.querySelector('.my-element')?.textContent.trim()"
```

### 8. Wait Then Act
```bash
thinkbrowse navigate "https://app.example.com"
thinkbrowse wait "button[type=submit]" --timeout 10000
thinkbrowse click "button[type=submit]"
```

### 9. Network Monitoring (Instead of Async Evaluate)
```bash
# evaluate can't await Promises — use network monitoring instead
thinkbrowse network    # See all requests the page made
```

### 10. Tab Switching (Local Mode)
```bash
thinkbrowse tabs
thinkbrowse attach <newTabId>
sleep 2                                    # Wait for connection
thinkbrowse screenshot --output /tmp/tab.png
```

### 11. Selector Scroll (Into View)
```bash
thinkbrowse evaluate "document.querySelector('.target-element')?.scrollIntoView({behavior:'smooth',block:'center'})"
```

### 12. Multi-Agent Coordination
```bash
thinkbrowse navigate "https://site-a.com" --tab 12345
thinkbrowse navigate "https://site-b.com" --tab 67890
```

### 13. Shadow DOM Traversal (Reddit, YouTube, Modern SPAs)

Standard `click`/`extract` can't reach elements inside Shadow DOM. Use `evaluate` with recursive traversal:

**Find and click by text inside Shadow DOM:**
```bash
# Write script to temp file to avoid shell quoting issues
cat > /tmp/shadow-click.js << 'JSEOF'
(function(){
  function findInShadow(root){
    var els=root.querySelectorAll("*");
    for(var i=0;i<els.length;i++){
      var el=els[i];
      if(el.shadowRoot){var r=findInShadow(el.shadowRoot);if(r)return r}
      var text=(el.textContent||"").trim();
      if(text==="TARGET_TEXT"&&el.children.length===0){
        var click=el;var p=el.parentElement;
        while(p){if(p.tagName==="A"||p.tagName==="BUTTON"||p.getAttribute("role")==="button"){click=p;break}p=p.parentElement}
        click.click();return "Clicked: "+click.tagName;
      }
    }
    return null;
  }
  return findInShadow(document)||"not found";
})()
JSEOF
SCRIPT=$(sed 's/TARGET_TEXT/Join/g' /tmp/shadow-click.js | tr '\n' ' ')
thinkbrowse evaluate "$SCRIPT"
```

**List all interactive elements inside Shadow DOM:**
```bash
thinkbrowse evaluate "(function(){function f(r,res){var els=r.querySelectorAll('button,a,[role=button]');for(var i=0;i<els.length;i++){var t=(els[i].textContent||'').trim();if(t.length>0&&t.length<50)res.push(t)}var all=r.querySelectorAll('*');for(var i=0;i<all.length;i++){if(all[i].shadowRoot)f(all[i].shadowRoot,res)}return res}return f(document,[]).slice(0,20)})()"
```

**Set input value inside Shadow DOM (React-compatible):**
```bash
cat > /tmp/shadow-fill.js << 'JSEOF'
(function(){
  function findInShadow(root){
    var els=root.querySelectorAll("textarea,input");
    for(var i=0;i<els.length;i++){
      if(els[i].getBoundingClientRect().height>0 && els[i].placeholder==="TARGET_PLACEHOLDER")return els[i];
    }
    var all=root.querySelectorAll("*");
    for(var i=0;i<all.length;i++){if(all[i].shadowRoot){var r=findInShadow(all[i].shadowRoot);if(r)return r}}
    return null;
  }
  var el=findInShadow(document);
  if(!el)return "not found";
  var proto=el.tagName==="TEXTAREA"?HTMLTextAreaElement.prototype:HTMLInputElement.prototype;
  var setter=Object.getOwnPropertyDescriptor(proto,"value").set;
  setter.call(el,"NEW_VALUE");
  el.dispatchEvent(new Event("input",{bubbles:true}));
  el.dispatchEvent(new Event("change",{bubbles:true}));
  return "set: "+el.value;
})()
JSEOF
SCRIPT=$(cat /tmp/shadow-fill.js | tr '\n' ' ')
thinkbrowse evaluate "$SCRIPT"
```

**Works for:** reading content, clicking buttons/links, navigating, extracting text, toggling settings.
**Fails for:** Reddit comment posting, some anti-bot form submissions — these require genuine browser-level user gestures that programmatic clicks can't satisfy. Use handoff for these cases.

## Anti-Patterns

| # | Anti-Pattern | Fix |
|---|-------------|-----|
| 1 | Generic CSS: `click "button"` | Specific: `click "button[type=submit]"` or click-by-text via evaluate |
| 2 | Text as selector: `click "Timeline & Logs"` | Use evaluate: `find(b=>b.textContent.includes('Timeline'))` |
| 3 | `fill` on React inputs | Use `type` — keyboard events trigger onChange |
| 4 | Async evaluate: `evaluate "fetch(...).then(...)"` | Returns `{}`. Use `thinkbrowse network` instead |
| 5 | Screenshot right after attach | `sleep 2` first — connection needs time |
| 6 | `document.body.innerText` | Target specific selectors, `.slice(0,2000)` |
| 7 | Not cleaning up sessions | Always `release` (local) or `cloud stop` (cloud) |

## Gotchas Reference

| Gotcha | Details |
|--------|---------|
| `url` returns plain string | `{"data": "https://..."}` not `{"data": {"url": "..."}}` |
| Screenshot returns path | `{"data": {"path": "/tmp/x.png"}}` — use Read tool to view |
| `evaluate` auto-wraps | Single expressions OK. Multi-statement needs IIFE: `(()=>{...})()` |
| Click ≠ navigation | Click returns immediately. Wait for new page element after clicking links |
| Scroll no-op on fixed layouts | Screenshot after scrolling to verify it worked |
| CSRF forms | Navigate → `sleep 2` → fill → submit. Token fetch must complete first |
| Protected iframes | Stripe/OAuth return `blocked: true, handoffRequired: true` — stop automation |
| Tab ownership persists | Crashed agent leaves stale lock. Use different tab or have original release |
| `switch-tab` ≠ `attach` | switch-tab = Chrome visual focus. attach = CLI command routing. Need both |
| Scroll syntax varies | CLI: `--direction down --amount 500`. browse.sh: `--down 500` |
