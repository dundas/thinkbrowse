# ThinkBrowse MCP — Complete Tool Reference (35 tools)

## Session Management (7)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `session_create` | Create browser session | `url?`, `thought` |
| `session_status` | Check session state | `sessionId`, `thought` |
| `session_delete` | Terminate session | `sessionId`, `thought` |
| `session_list` | List all sessions | `thought` |
| `session_wait_ready` | Poll until provisioned | `sessionId`, `thought` |
| `session_use` | Set default session | `sessionId`, `thought` |
| `session_artifacts` | List screenshots/recordings | `sessionId`, `thought` |

## Mode Switching (1)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `set_mode` | Switch local/cloud/auto at runtime | `mode: local|cloud|auto`, `thought` |

## Navigation (4)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `navigate` | Go to URL | `url`, `thought` |
| `go_back` | Browser back | `thought` |
| `go_forward` | Browser forward | `thought` |
| `wait_for_element` | Wait for CSS selector | `selector`, `timeout?`, `thought` |

## Interaction (7)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `click` | Click element | `selector`, `thought` |
| `type_text` | Type with keyboard events (React-safe) | `selector`, `text`, `thought` |
| `fill` | Set value directly (plain HTML only) | `selector`, `value`, `thought` |
| `press_key` | Press key | `key` (Enter, Tab, Escape, ArrowDown), `thought` |
| `scroll` | Scroll page | `direction` (up/down), `amount?`, `thought` |
| `hover` | Hover over element | `selector`, `thought` |
| `select_option` | Select dropdown option | `selector`, `value`, `thought` |

## Observation (4)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `snapshot` | Accessibility tree (lightweight, best for AI) | `thought` |
| `screenshot` | Visual capture (base64 image) | `thought` |
| `extract` | Extract text/attributes from elements | `selector`, `thought` |
| `evaluate` | Execute JavaScript on page | `script`, `thought` |

## Tabs — Local Only (3)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `tab_list` | List all Chrome tabs | `thought` |
| `tab_new` | Open new tab | `url?`, `thought` |
| `tab_close` | Close a tab | `tabId`, `thought` |

## Dialog Handling (2)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `get_dialog` | Check for pending dialog | `thought` |
| `handle_dialog` | Accept/dismiss dialog | `action` (accept/dismiss), `text?`, `thought` |

## Wait (1)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `wait_for_text` | Wait for text on page | `text`, `timeout?`, `thought` |

## Monitoring (3)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `console_messages` | Browser console logs | `thought` |
| `network_requests` | Network request log | `thought` |
| `clear_logs` | Reset console/network | `thought` |

## Tasks — Cloud Only (3)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `task_create` | Create task plan from text | `task`, `thought` |
| `task_execute` | Execute a task plan | `planId`, `thought` |
| `task_status` | Check task execution progress | `taskId`, `thought` |

## Mode Availability

| Category | Local | Cloud |
|----------|:-----:|:-----:|
| Session management | Y | Y |
| Navigation | Y | Y |
| Interaction | Y | Y |
| Observation | Y | Y |
| Dialog handling | Y | Y |
| Wait | Y | Y |
| Monitoring | Y | Y |
| **Tabs** | **Y** | **-** |
| **Tasks** | **-** | **Y** |
