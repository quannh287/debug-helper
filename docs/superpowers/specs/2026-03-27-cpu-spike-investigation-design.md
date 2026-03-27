# CPU Spike Investigation Design

**Date:** 2026-03-27
**Status:** Approved
**Problem:** After stopping debug-helper recording and switching to other tasks (30s–2min later), Chrome CPU usage spikes to ~200%, making the browser unusable. Consistently reproduced on localhost:3000 with HMR active.

---

## Root Cause Hypotheses

Three suspects identified from static code analysis, in order of likelihood:

### Suspect A — `setInterval` never cleared (service-worker.js:358)

```js
setInterval(() => SW.flushBuffer(), SW.FLUSH_INTERVAL); // runs forever
```

This interval fires every 2 seconds and is never cancelled — no `clearInterval` call exists anywhere. It keeps the service worker alive and calls `flushBuffer()` → `Storage.getCurrentSession()` (async chrome.storage lookup) indefinitely after stop.

### Suspect B — Console/network interceptors never deactivate (console-capture.js, network-capture.js)

Both scripts patch `console.*`, `fetch`, and `XHR` once on injection and have no stop mechanism. After `session:stop`, they continue intercepting every console log and network request on the page, calling `window.postMessage()` for each one. On localhost:3000 with HMR, this is a high-frequency stream (dozens of events per hot reload cycle).

### Suspect C — bridge.js message relay never stops (bridge.js:7)

```js
window.addEventListener('message', (e) => { ... chrome.runtime.sendMessage(msg) ... });
```

This listener is never removed. It relays every message from Suspect B to the service worker, where each one triggers `bufferEvent()` → `Storage.getCurrentSession()`. Combined with HMR activity, this creates a continuous message storm.

**Combined effect:** HMR hot reload → console/network events → bridge relay → service worker message handler → async storage lookup, repeating continuously. The setInterval compounds this by keeping the SW alive and adding 2-second flushBuffer calls on top.

---

## Investigation Plan

### Phase 1 — Measure (Chrome DevTools Profiler)

**Goal:** Confirm the service worker is consuming CPU and identify the hottest call sites.

**Steps:**
1. Open `chrome://extensions` → Debug Helper → click **"Service Worker"** link to open SW DevTools
2. Also open **Chrome Task Manager** (`Shift+Esc`) — keep visible during the test
3. In SW DevTools → **Performance** tab → start recording
4. Start a debug-helper session on localhost:3000, use the app for ~1 minute, stop the session
5. Wait 1–2 minutes (the duration where CPU spike appears)
6. Stop the profiler

**Evidence to capture:**
- Flame chart: identify hottest functions by self-time
- Call frequency of `flushBuffer`, `bufferEvent`, `handleMessage`, `Storage.getCurrentSession`
- Chrome Task Manager CPU % for the extension row over time

**Success criteria:** Can name which function(s) account for >50% of CPU time.

---

### Phase 2 — Confirm Root Cause (Instrumented Logging)

**Goal:** Measure exact call frequency of each suspect after stop.

**Temporary instrumentation to add:**

In `background/service-worker.js` — `flushBuffer()`:
```js
async flushBuffer() {
  console.count('[DH] flushBuffer called');
  if (this.eventBuffer.length === 0) return;
  // ... rest unchanged
}
```

In `background/service-worker.js` — `bufferEvent()`:
```js
async bufferEvent(event) {
  console.count('[DH] bufferEvent: ' + event.type);
  const session = await Storage.getCurrentSession();
  // ... rest unchanged
}
```

In `content/bridge.js` — message listener:
```js
window.addEventListener('message', (e) => {
  if (e.source !== window) return;
  if (!e.data || e.data.source !== 'debug-helper-main') return;
  console.count('[DH] bridge relay: ' + e.data.type);
  // ... rest unchanged
});
```

**Observation:** Watch SW DevTools console after stop. Expected findings:
- `[DH] flushBuffer called` increments every ~2 seconds indefinitely → confirms Suspect A
- `[DH] bridge relay: event:console` and `event:network` increment on every HMR reload → confirms Suspects B+C
- `[DH] bufferEvent` count growing → confirms message storm reaching SW

**Success criteria:** Can quantify events/minute for each suspect.

---

### Phase 3 — Targeted Fix

Applied only after Phase 1+2 confirm which suspects are active.

| Confirmed Suspect | Fix |
|-------------------|-----|
| A — setInterval never cleared | Save interval ID: `this._flushInterval = setInterval(...)`. Call `clearInterval(this._flushInterval)` in `stopKeepalive()` |
| B — Console/network interceptors never stop | Add `recording` flag + `stopCapture()` function in each script. Send `recording:stop` message to trigger deactivation |
| C — bridge.js never removes listener | Extract handler to named function; call `window.removeEventListener` when SW sends `recording:stop` |

**Validation:** After fix, repeat Phase 1 profiling. CPU should not accumulate after stop on a busy HMR page.

---

## Constraints & Scope

- Instrumentation in Phase 2 is temporary — must be removed before any release
- Fix scope is limited to the three suspects; no broader refactoring
- The `DevReload` polling loop (service-worker.js:363–416) is a separate concern and is already commented out — do not touch it

---

## Files Involved

| File | Role in investigation |
|------|-----------------------|
| `background/service-worker.js` | Suspects A (setInterval) + message handler; Phase 2 instrumentation |
| `content/bridge.js` | Suspect C; Phase 2 instrumentation |
| `content/console-capture.js` | Suspect B — console intercept |
| `content/network-capture.js` | Suspect B — fetch/XHR intercept |
