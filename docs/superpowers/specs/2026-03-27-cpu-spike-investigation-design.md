# CPU Spike Investigation Design

**Date:** 2026-03-27
**Status:** Draft
**Problem:** After stopping debug-helper recording and switching to other tasks (30s–2min later), Chrome CPU usage spikes to ~200%, making the browser unusable. Consistently reproduced on localhost:3000 with HMR active.

---

## Root Cause Hypotheses

Three suspects identified from static code analysis, in order of combined impact:

### Suspect A — `setInterval` never cleared (service-worker.js:358)

```js
setInterval(() => SW.flushBuffer(), SW.FLUSH_INTERVAL); // no clearInterval anywhere
```

**Important nuance:** In MV3, `setInterval` does NOT keep a service worker alive — the browser terminates it regardless. The actual keepalive mechanism is `chrome.alarms` (see `startKeepalive()` at line 331), which fires every 24 seconds and calls `flushBuffer()`. This alarm IS already cleared in `stopKeepalive()` via `chrome.alarms.clear()`, which IS called in `stopSession()` at line 150. So the alarm-based keepalive appears to be working correctly.

The `setInterval` is a secondary problem: it fires only while the SW happens to be awake (e.g., when handling a message from Suspects B/C). It does not independently keep the SW alive, but it does add extra `flushBuffer()` calls on each wakeup.

Additionally, `flushBuffer()` has an early-exit guard — it returns immediately if `this.eventBuffer.length === 0` without touching storage. The cost is only real when the buffer is non-empty, which requires Suspects B/C to be generating events.

**Severity:** Secondary — amplifier of Suspects B+C, not standalone root cause.

### Suspect B — Console/network interceptors never deactivate (console-capture.js, network-capture.js)

Both scripts patch `console.*`, `fetch`, and `XHR` once on injection and have no stop mechanism. After `session:stop`, they continue intercepting every console log and network request on the page, calling `window.postMessage()` for each one. On localhost:3000 with HMR, this is a high-frequency stream (dozens of events per hot reload cycle).

**Severity:** High — primary event generator.

### Suspect C — bridge.js message relay never stops (bridge.js:7)

```js
window.addEventListener('message', (e) => { ... chrome.runtime.sendMessage(msg) ... });
```

This listener is never removed. It relays every message from Suspect B to the service worker, where each one triggers `bufferEvent()` → `Storage.getCurrentSession()` (async chrome.storage lookup).

**Additional risk — HMR double-registration:** bridge.js has an idempotency guard:
```js
if (window.__debugHelperBridge) {
  try { if (chrome.runtime?.id) return; } catch {}
}
```
If the service worker is terminated (making `chrome.runtime` unavailable), the check can fail and a second listener is registered on the next HMR page reload. This means the message storm can be multiplied by the number of HMR reloads since the recording was stopped.

**Severity:** High — primary relay of Suspect B's flood to the service worker.

**Combined effect:** HMR hot reload → console/network events (B) → bridge relay, potentially duplicated (C) → service worker message handler → async storage lookup, repeatedly. The setInterval (A) adds extra flushBuffer calls during each SW wakeup.

---

## Investigation Plan

### Phase 1 — Measure (Chrome DevTools Profiler)

**Goal:** Confirm the service worker is consuming CPU and identify the hottest call sites.

**Steps:**
1. Open `chrome://extensions` → Debug Helper → click **"service worker"** link (labeled "Inspect views: service worker" in some Chrome versions) to open SW DevTools
2. Also open **Chrome Task Manager** (`Shift+Esc`) — keep visible during the test
3. In SW DevTools → **Performance** tab → start recording
4. Start a debug-helper session on localhost:3000, use the app for ~1 minute, stop the session
5. Wait 1–2 minutes (the duration where CPU spike appears)
6. Stop the profiler

**Evidence to capture:**
- Flame chart: identify hottest functions by self-time
- Call frequency of `flushBuffer`, `bufferEvent`, `handleMessage`, `Storage.getCurrentSession`
- Chrome Task Manager CPU % for the extension row over time
- Whether the SW is being woken up at all after stop (if Task Manager shows 0% CPU, the SW is already terminating correctly and the issue is elsewhere)

**Success criteria:** Can name which function(s) account for >50% of CPU time.

---

### Phase 2 — Confirm Root Cause (Instrumented Logging)

**Goal:** Measure exact call frequency of each suspect after stop.

**Temporary instrumentation to add:**

In `background/service-worker.js` — at the top of `flushBuffer()` (line 204), before the early-exit guard:
```js
async flushBuffer() {
  console.count('[DH] flushBuffer called'); // ADD THIS LINE
  if (this.eventBuffer.length === 0) return;
  // ... rest unchanged
}
```

In `background/service-worker.js` — at the top of `bufferEvent()`:
```js
async bufferEvent(event) {
  console.count('[DH] bufferEvent: ' + event.type); // ADD THIS LINE
  const session = await Storage.getCurrentSession();
  // ... rest unchanged
}
```

In `content/bridge.js` — after the two guard clauses (line 9), before `const msg = { ...e.data }`:
```js
window.addEventListener('message', (e) => {
  if (e.source !== window) return;
  if (!e.data || e.data.source !== 'debug-helper-main') return;
  console.count('[DH] bridge relay: ' + e.data.type); // ADD THIS LINE — before const msg line
  const msg = { ...e.data };
  // ... rest unchanged
});
```

**Observation:** Watch SW DevTools console after stop. Expected findings:
- `[DH] flushBuffer called` increments → confirms SW is still being woken up (A)
- `[DH] bridge relay: event:console` / `event:network` counting up on every HMR reload → confirms Suspects B+C
- Large relay count (e.g., 5–10x expected) → suggests bridge double-registration from HMR reloads
- `[DH] bufferEvent` count growing → confirms message storm reaching SW

**Success criteria:** Can quantify events/minute for each suspect.

---

### Phase 3 — Targeted Fix

Applied only after Phase 1+2 confirm which suspects are active.

| Confirmed Suspect | Fix |
|-------------------|-----|
| A — setInterval never cleared | Save ID: `this._flushInterval = setInterval(...)`. Call `clearInterval(this._flushInterval)` in `stopSession()` alongside `stopKeepalive()` |
| B — Console/network interceptors never stop | Add `recording` flag + `stopCapture()` in each script. **Note:** `console-capture.js` and `network-capture.js` run in the MAIN world and cannot receive `chrome.tabs.sendMessage` directly (that goes to ISOLATED world only). The relay path needed: `stopSession()` sends `recording:stop` via `chrome.tabs.sendMessage` → `recorder.js` (ISOLATED) listens and forwards into MAIN world via `window.postMessage` → capture scripts listen for the postMessage and call `stopCapture()` |
| C — bridge.js relay never stops + double registration | Extract listener to named function; call `window.removeEventListener` on `recording:stop`; fix idempotency guard to not rely on `chrome.runtime` availability check |

**Validation:** After fix, repeat Phase 1 profiling. CPU should not accumulate after stop on a busy HMR page.

---

## Constraints & Scope

- Instrumentation in Phase 2 is temporary — must be removed before any release
- Fix scope is limited to the three suspects; no broader refactoring
- The `DevReload` polling loop (service-worker.js:363–416) is already commented out — do not touch it
- Verify `stopKeepalive()` is being called correctly on stop before implementing Suspect A fix — current code already calls it at line 150

---

## Files Involved

| File | Role in investigation |
|------|-----------------------|
| `background/service-worker.js` | Suspect A (setInterval) + message handler; Phase 2 instrumentation |
| `content/bridge.js` | Suspect C; Phase 2 instrumentation; HMR double-registration risk |
| `content/console-capture.js` | Suspect B — console intercept |
| `content/network-capture.js` | Suspect B — fetch/XHR intercept |
