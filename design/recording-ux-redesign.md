# Recording UX Redesign: Unified Capture Flow

## Problem

The current recording UI has a **Tab / Desktop toggle** that creates confusion:
- **Tab mode** requires `activeTab` permission (only granted by clicking the extension icon), so it fails from the sidepanel
- **Desktop mode** uses `getDisplayMedia()` which works everywhere but has no DOM/network observability
- Users don't understand why tab recording fails on certain pages

## Insight

The capture mode should be determined by **what the user picks in Chrome's share picker**, not by a manual toggle. Chrome's `getDisplayMedia()` already provides `displaySurface` metadata that tells us what was selected.

---

## Current Flow (Before)

### Sidepanel — Record Tab (Idle)

```
┌─────────────────────────────────┐
│  ThinkBrowse          ⚙️        │
├──────────┬──────────────────────┤
│  Agent   │  Record  ◄──active   │
├──────────┴──────────────────────┤
│                                 │
│  CAPTURE MODE                   │
│  ┌──────────┐ ┌──────────┐     │
│  │ 🌐       │ │ 🖥️       │     │
│  │ This Tab │ │ Desktop  │     │
│  │ Current  │ │ Entire   │     │
│  │ browser  │ │ screen   │     │
│  │ tab      │ │          │     │
│  └──────────┘ └──────────┘     │
│                                 │
│  (Tab mode: feature toggles)    │
│  ☑ Microphone  ☑ DOM           │
│  ☑ Console     ☑ Network       │
│                                 │
│  ┌─────────────────────────┐   │
│  │    Start Recording      │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ Ready to record    Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Starting State

```
┌─────────────────────────────────┐
│  CAPTURE MODE (hidden)          │
│                                 │
│  ┌─────────────────────────┐   │
│  │ Waiting for selection...│   │
│  │ Pick a tab, window, or  │   │
│  │ screen to record        │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ ● Waiting...       Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Recording

```
┌─────────────────────────────────┐
│                                 │
│        ● Recording              │
│                                 │
│         01:23                   │
│                                 │
│  ┌─────────────────────────┐   │
│  │    Stop Recording       │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ ● 01:23            Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Done

```
┌─────────────────────────────────┐
│                                 │
│  Recording done    Record again │
│  What would you like to do?     │
│                                 │
│  ┌───────────┐ ┌───────────┐   │
│  │ Save local│ │Share w/ AI│   │
│  └───────────┘ └───────────┘   │
│                                 │
│  (red if upload failed)         │
│  Upload failed — click Share    │
│  with AI to retry               │
│                                 │
├─────────────────────────────────┤
│ Ready to record    Credits: 9867│
└─────────────────────────────────┘
```

### Popup — Idle

```
┌──────────────────────────────────┐
│ 🟧 ThinkBrowse    ● Ready   ⚙️  │
├──────────────────────────────────┤
│ ● Local API connected            │
├──────────────────────────────────┤
│ QUICK ACTIONS                    │
│ [Extract] [Summarize] [Screenshot│
├──────────────────────────────────┤
│ Simple | Power | Developer       │
│                                  │
│ [Tab] | [Desktop]                │
│                                  │
│ Intent: [Bug Report / Feedback ▼]│
│                                  │
│ ☑ Mic  ☑ DOM  ☑ Console ☑ Net  │
│                                  │
│ ⚠️ Navigate to a website first   │
│ (on chrome:// pages)             │
│                                  │
│ ┌────────────────────────────┐  │
│ │  ● Start Recording         │  │
│ └────────────────────────────┘  │
├──────────────────────────────────┤
│ ▶ Debug Logs (3)                 │
└──────────────────────────────────┘
```

---

## Proposed Flow (After)

### Key Changes

1. **Remove Tab/Desktop toggle** — single "Start Recording" button
2. **Chrome share picker decides the mode** — user picks tab, window, or screen
3. **Auto-detect capture type** via `displaySurface` from the stream:
   - `"browser"` → tab was selected → inject DOM/network/console interceptors
   - `"window"` or `"monitor"` → screen/window → video-only mode
4. **Feature toggles shown after stream acquired** (for tab capture) — not before
5. **Works from sidepanel** — no `activeTab` dependency (uses `getDisplayMedia` for all)

### Sidepanel — Record Tab (Idle)

```
┌─────────────────────────────────┐
│  ThinkBrowse          ⚙️        │
├──────────┬──────────────────────┤
│  Agent   │  Record  ◄──active   │
├──────────┴──────────────────────┤
│                                 │
│  Record your screen, a window,  │
│  or a browser tab.              │
│                                 │
│  ☑ Include microphone           │
│                                 │
│  ┌─────────────────────────┐   │
│  │    Start Recording      │   │
│  └─────────────────────────┘   │
│                                 │
│  Tab recordings include DOM,    │
│  console, and network capture   │
│  automatically.                 │
│                                 │
├─────────────────────────────────┤
│ Ready to record    Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Starting (Share Picker Open)

```
┌─────────────────────────────────┐
│                                 │
│  ┌─────────────────────────┐   │
│  │ Choose what to share... │   │
│  │                         │   │
│  │ Chrome will show a      │   │
│  │ picker. Select a tab,   │   │
│  │ window, or screen.      │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ ● Waiting...       Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Recording (Tab Detected)

```
┌─────────────────────────────────┐
│                                 │
│     ● Recording — Tab           │
│     github.com/dundas/seedid    │
│                                 │
│         01:23                   │
│                                 │
│  Capturing: video, DOM,         │
│  console, network               │
│                                 │
│  ┌─────────────────────────┐   │
│  │    Stop Recording       │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ ● 01:23            Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Recording (Screen/Window Detected)

```
┌─────────────────────────────────┐
│                                 │
│     ● Recording — Screen        │
│                                 │
│         01:23                   │
│                                 │
│  Capturing: video only          │
│                                 │
│  ┌─────────────────────────┐   │
│  │    Stop Recording       │   │
│  └─────────────────────────┘   │
│                                 │
├─────────────────────────────────┤
│ ● 01:23            Credits: 9867│
└─────────────────────────────────┘
```

### Sidepanel — Done (unchanged)

```
┌─────────────────────────────────┐
│                                 │
│  Recording done    Record again │
│  What would you like to do?     │
│                                 │
│  ┌───────────┐ ┌───────────┐   │
│  │ Save local│ │Share w/ AI│   │
│  └───────────┘ └───────────┘   │
│                                 │
├─────────────────────────────────┤
│ Ready to record    Credits: 9867│
└─────────────────────────────────┘
```

### Popup — Idle (Simplified)

```
┌──────────────────────────────────┐
│ 🟧 ThinkBrowse    ● Ready   ⚙️  │
├──────────────────────────────────┤
│ ● Local API connected            │
├──────────────────────────────────┤
│ QUICK ACTIONS                    │
│ [Extract] [Summarize] [Screenshot│
├──────────────────────────────────┤
│                                  │
│ Record your screen, a window,    │
│ or a browser tab.                │
│                                  │
│ ☑ Include microphone             │
│                                  │
│ ┌────────────────────────────┐  │
│ │  ● Start Recording         │  │
│ └────────────────────────────┘  │
│                                  │
│ Tab recordings include DOM,      │
│ console, and network capture.    │
│                                  │
├──────────────────────────────────┤
│ ▶ Debug Logs (3)                 │
└──────────────────────────────────┘
```

---

## Technical Implementation

### Stream Type Detection

After `getDisplayMedia()` returns, check `displaySurface`:

```typescript
const stream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true });
const videoTrack = stream.getVideoTracks()[0];
const settings = videoTrack.getSettings();
const surface = settings.displaySurface; // 'browser' | 'window' | 'monitor'

chrome.runtime.sendMessage({
  type: 'STREAM_ACQUIRED',
  payload: { displaySurface: surface }
});
```

### Service Worker State Changes

```typescript
// In STREAM_ACQUIRED handler:
if (payload.displaySurface === 'browser') {
  // Tab was selected — inject interceptors
  captureMode = 'tab';
  // Find the tab that matches (Chrome provides tab ID in some cases)
  // Inject content script for DOM/network/console capture
} else {
  // Window or screen — video only
  captureMode = 'desktop';
}

currentState = {
  status: 'recording',
  startTime: Date.now(),
  captureMode, // 'tab' | 'desktop' — auto-detected, not user-selected
};
```

### What Gets Removed

- `CaptureMode` toggle in sidepanel (`recordingConfig.captureMode`)
- `CaptureMode` toggle in popup (`config.captureMode`)
- Feature toggles grid (DOM/Console/Network) — always on for tab, N/A for desktop
- `isRestrictedPage` check — not needed since `getDisplayMedia` works everywhere
- `tabCapture.getMediaStreamId()` code path — replaced by `getDisplayMedia` for all

### What Gets Added

- `displaySurface` detection in offscreen document
- `STREAM_ACQUIRED` payload includes `displaySurface`
- Recording state includes `captureMode` (auto-detected)
- UI shows what's being captured (tab name vs "Screen")
- Microphone toggle (only pre-recording setting)

### Migration Notes

- The `RecordingConfig.captureMode` field is still stored but auto-set
- `captureMode: 'tab'` in config means "user picked a browser tab in the picker"
- `captureMode: 'desktop'` means "user picked a window or screen"
- Feature toggles (DOM, console, network) are always enabled for tab captures
- Existing recordings are not affected

---

## Active Tab Tracking (Tab Capture)

When `displaySurface === 'browser'`, instead of trying to identify which specific tab was selected in the picker, **follow the active tab**:

### How it works

1. **`STREAM_ACQUIRED` with `displaySurface: 'browser'`** → query the active tab → set as `recordingTabId` → inject interceptors
2. **`chrome.tabs.onActivated`** → user switched tabs → update `recordingTabId` → inject interceptors into new tab → log the tab switch
3. **`chrome.tabs.onUpdated`** → active tab navigated → re-inject interceptors (already built)
4. **Recording stop** → clean up `onActivated` listener

### What gets captured across tab switches

```
User flow:                    What ThinkBrowse captures:
────────────                  ──────────────────────────
github.com (start)      →    DOM + console + network + video
  ↓ switch tab
slack.com               →    DOM + console + network + video
  ↓ switch tab
github.com (back)       →    DOM + console + network + video
  ↓ stop recording
```

### Service worker changes

```typescript
// New: track tab switches during recording
let onActivatedListener: ((info: chrome.tabs.TabActiveInfo) => void) | null = null;

// In STREAM_ACQUIRED handler (when displaySurface === 'browser'):
onActivatedListener = (activeInfo) => {
  if (currentState.status !== 'recording') return;

  const newTabId = activeInfo.tabId;
  recordingTabId = newTabId;

  chrome.tabs.get(newTabId, (tab) => {
    if (tab.url && !tab.url.startsWith('chrome://')) {
      recordingUrl = tab.url;
      recordingTitle = tab.title || '';
      visitedUrls.push({ url: tab.url, title: tab.title || '', timestamp: Date.now() - startTime });
      injectInterceptors(newTabId).catch(() => {});
    }
  });
};
chrome.tabs.onActivated.addListener(onActivatedListener);

// In stopRecording:
if (onActivatedListener) {
  chrome.tabs.onActivated.removeListener(onActivatedListener);
  onActivatedListener = null;
}
```

### Why this is better than identifying the picked tab

| Approach | Pros | Cons |
|----------|------|------|
| Identify picked tab | Captures exactly what user chose | `getDisplayMedia` doesn't expose tab ID; fragile matching |
| Follow active tab | Simple, reliable, captures tab switches | May inject into tabs user didn't intend to record |

Following the active tab is what the user expects — they're looking at it, they're interacting with it, that's what they want recorded.

---

## Open Questions

1. **Mic permission**: Still need the permission tab flow for mic access from offscreen. Keep the existing pattern.

2. **Profile tabs (Simple/Power/Developer)**: These only exist in the popup. Remove or keep? They control feature toggles which are now automatic. **Recommendation**: Remove — simplify to one flow.

3. **Intent dropdown**: Keep in popup? It's useful metadata for AI analysis but adds friction. **Recommendation**: Move to post-recording (when sharing with AI) instead of pre-recording.

4. **`displaySurface` browser support**: `getSettings().displaySurface` is available in Chrome 107+. Need to verify it's reliable. Fallback: treat unknown surface as desktop mode.
