# CPU Spike Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Diagnose and fix the CPU spike that occurs after stopping debug-helper recording on a busy HMR page.

**Architecture:** Phase 1–2 add temporary instrumentation to measure call frequency of suspects, Phase 3 applies targeted fixes to silence content scripts when not recording and prevent bridge double-registration.

**Tech Stack:** Vanilla JS, Chrome Extension MV3, no build step — reload extension via `chrome://extensions` after every code change.

---

> **How to reload the extension after each change:**
> `chrome://extensions` → Debug Helper → click the refresh icon (↺). Then reload the page under test (`localhost:3000`).

---

## File Map

| File | Changes |
|------|---------|
| `background/service-worker.js` | Task 1 (instrumentation) + Task 5 (clearInterval fix) + Task 7 (remove instrumentation) |
| `content/bridge.js` | Task 1 (instrumentation) + Task 6 (guard fix) + Task 7 (remove instrumentation) |
| `content/console-capture.js` | Task 4 (recording flag + stop relay listener) |
| `content/network-capture.js` | Task 4 (recording flag + stop relay listener) |
| `content/recorder.js` | Task 4 (relay recording:start/stop to MAIN world) |

---

## Task 1: Add Phase 2 Instrumentation

**Files:**
- Modify: `background/service-worker.js` (lines 204, 188)
- Modify: `content/bridge.js` (line 10)

- [ ] **Step 1: Add counter to `flushBuffer()` in service-worker.js**

  Open `background/service-worker.js`. Find `async flushBuffer()` at line 204. Insert one line after the opening brace, before the early-exit guard:

  ```js
  async flushBuffer() {
    console.count('[DH] flushBuffer called'); // TEMP instrumentation
    if (this.eventBuffer.length === 0) return;
    // ... rest unchanged
  ```

- [ ] **Step 2: Add counter to `bufferEvent()` in service-worker.js**

  In the same file, find `async bufferEvent(event)` at line 188. Insert one line after the opening brace:

  ```js
  async bufferEvent(event) {
    console.count('[DH] bufferEvent: ' + event.type); // TEMP instrumentation
    const session = await Storage.getCurrentSession();
    // ... rest unchanged
  ```

- [ ] **Step 3: Add counter to bridge relay in bridge.js**

  Open `content/bridge.js`. Find the message listener body. Insert the counter immediately before `const msg = { ...e.data }` (that line is currently at line 11 in the file — insert your new line above it):

  ```js
  window.addEventListener('message', (e) => {
    if (e.source !== window) return;
    if (!e.data || e.data.source !== 'debug-helper-main') return;
    console.count('[DH] bridge relay: ' + e.data.type); // TEMP instrumentation
    const msg = { ...e.data };
    delete msg.source;
    chrome.runtime.sendMessage(msg).catch(() => {});
  });
  ```

- [ ] **Step 4: Reload extension and verify instrumentation loads**

  Reload extension at `chrome://extensions`. Open `chrome://extensions` → Debug Helper → **"service worker"** (inspect link). In the SW DevTools Console, start a recording and click around the test page. You should see `[DH] flushBuffer called`, `[DH] bufferEvent: event:dom`, and `[DH] bridge relay: event:console` counting up.

- [ ] **Step 5: Commit instrumentation**

  ```bash
  git add background/service-worker.js content/bridge.js
  git commit -m "temp: add instrumentation for CPU spike diagnosis (to be removed)"
  ```

---

## Task 2: Run Phase 1 Profiler (Measure)

**No code changes — manual measurement steps.**

- [ ] **Step 1: Open SW DevTools Performance profiler**

  1. Go to `chrome://extensions` → Debug Helper → click **"service worker"** (under "Inspect views")
  2. In the SW DevTools window: click **Performance** tab → click the record button (●)

- [ ] **Step 2: Open Chrome Task Manager**

  Press `Shift+Esc` in Chrome. Locate the "Extension: Debug Helper" row. Keep this window visible.

- [ ] **Step 3: Run the test scenario**

  1. Navigate to `localhost:3000`
  2. Start a debug-helper recording session (via popup or `Cmd+Shift+R`)
  3. Let HMR do several hot reloads (save a file a few times)
  4. Stop the recording session
  5. Continue normal dev work for 1–2 minutes

- [ ] **Step 4: Stop the profiler and analyze**

  Click the stop button in SW DevTools Performance tab. In the flame chart:
  - Look for functions with high **Self Time**: `flushBuffer`, `bufferEvent`, `handleMessage`, `getCurrentSession`
  - Note whether activity continues after you stopped recording
  - Record the CPU% from Task Manager before and after stopping recording

- [ ] **Step 5: Record findings**

  Note in a scratch file or comment which functions are hottest and whether CPU is still high after stop.

---

## Task 3: Observe Phase 2 Console Counts (Confirm Root Cause)

**No code changes — observation of instrumentation added in Task 1.**

- [ ] **Step 1: Run the same scenario and watch SW console**

  Repeat the test scenario from Task 2. After stopping the recording, watch the SW DevTools **Console** tab:

  - `[DH] flushBuffer called` — if this increments after stop → SW is being woken up (Suspect A active)
  - `[DH] bridge relay: event:console` or `event:network` — if these increment on every HMR reload after stop → Suspects B+C active
  - `[DH] bridge relay` count that is 2× or 3× higher than expected → bridge double-registration (HMR bug in Suspect C)
  - `[DH] bufferEvent:` counting up → confirms message storm reaching SW

- [ ] **Step 2: Quantify**

  Note approximate counts over 60 seconds after stop. If `bridge relay` exceeds 50/min on a normal HMR page → primary root cause confirmed.

  **Expected confirmation:** `bridge relay` and `bufferEvent` spike → Suspects B+C are primary. `flushBuffer called` follows → Suspect A is secondary amplifier.

---

## Task 4: Fix Suspect B — Silence Capture Scripts When Not Recording

**Files:**
- Modify: `content/console-capture.js`
- Modify: `content/network-capture.js`
- Modify: `content/recorder.js`

The MAIN world scripts (console-capture.js, network-capture.js) cannot receive `chrome.tabs.sendMessage` directly — that only reaches ISOLATED world scripts. The relay path: recorder.js (ISOLATED) receives `recording:start/stop` → forwards via `window.postMessage` → MAIN scripts listen and toggle a `recording` flag.

- [ ] **Step 1: Add recording flag and stop listener to console-capture.js**

  Open `content/console-capture.js`. The current structure is a single IIFE with `window[PREFIX + 'consolePatched']` guard. Make these changes:

  After `window[PREFIX + 'consolePatched'] = true;`, add the recording flag:
  ```js
  let recording = false; // starts false; activated by recorder.js relay
  ```

  In the `post()` function, add an early-exit guard as the first line:
  ```js
  function post(level, args, stack) {
    if (!recording) return; // ADD THIS LINE
    window.postMessage({ ... }, '*');
  }
  ```

  > **Behavioral note:** `post()` is also called by the `error` and `unhandledrejection` window listeners at the bottom of the file. Adding this guard means those events are also silenced when not recording. This is intentional — there is no active session to store them in.

  At the bottom of the IIFE, before the closing `})();`, add the relay listener:
  ```js
  window.addEventListener('message', (e) => {
    if (e.source !== window) return;
    if (!e.data || e.data.source !== 'debug-helper-isolated') return;
    if (e.data.type === 'recording:start') recording = true;
    if (e.data.type === 'recording:stop') recording = false;
  });
  ```

- [ ] **Step 2: Add recording flag and stop listener to network-capture.js**

  Open `content/network-capture.js`. Same pattern:

  After `window[PREFIX + 'networkPatched'] = true;`, add:
  ```js
  let recording = false;
  ```

  In the `post()` function, add early-exit as first line:
  ```js
  function post(data) {
    if (!recording) return; // ADD THIS LINE
    window.postMessage({ source: 'debug-helper-main', type: 'event:network', ...data }, '*');
  }
  ```

  At the bottom of the IIFE, before `})();`, add:
  ```js
  window.addEventListener('message', (e) => {
    if (e.source !== window) return;
    if (!e.data || e.data.source !== 'debug-helper-isolated') return;
    if (e.data.type === 'recording:start') recording = true;
    if (e.data.type === 'recording:stop') recording = false;
  });
  ```

- [ ] **Step 3: Add postMessage relay in recorder.js**

  Open `content/recorder.js`. Replace `startRecording()` and `stopRecording()` in full (lines 171–189):
  ```js
  function startRecording() {
    window.postMessage({ source: 'debug-helper-isolated', type: 'recording:start' }, '*');
    recording = true;
    document.addEventListener('click', onClick, true);
    document.addEventListener('dblclick', onClick, true);
    document.addEventListener('input', onInput, true);
    document.addEventListener('change', onChange, true);
    document.addEventListener('submit', onSubmit, true);
    window.addEventListener('scroll', onScroll, true);
  }

  function stopRecording() {
    window.postMessage({ source: 'debug-helper-isolated', type: 'recording:stop' }, '*');
    recording = false;
    document.removeEventListener('click', onClick, true);
    document.removeEventListener('dblclick', onClick, true);
    document.removeEventListener('input', onInput, true);
    document.removeEventListener('change', onChange, true);
    document.removeEventListener('submit', onSubmit, true);
    window.removeEventListener('scroll', onScroll, true);
  }
  ```

- [ ] **Step 4: Reload extension and verify**

  Reload extension. Open localhost:3000. Start a recording — verify console logs and network requests appear in the side panel (capture is working). Stop recording — verify that in the SW console, `[DH] bridge relay: event:console` and `[DH] bridge relay: event:network` stop incrementing on subsequent HMR reloads.

  **Also verify page-refresh edge case:** While recording is active, save a file to trigger an HMR full page reload (or manually refresh localhost:3000). Confirm that console/network events continue to appear in the side panel after the refresh — this confirms `recorder.js` relays `recording:start` to the MAIN capture scripts on page restore.

- [ ] **Step 5: Commit**

  ```bash
  git add content/console-capture.js content/network-capture.js content/recorder.js
  git commit -m "fix: silence console/network capture scripts when not recording"
  ```

---

## Task 5: Fix Suspect A — Clear the setInterval on Stop

**Files:**
- Modify: `background/service-worker.js`

- [ ] **Step 1: Move setInterval into startKeepalive() and add clearInterval to stopKeepalive()**

  Find `startKeepalive()` at line 331 and `stopKeepalive()` at line 335. Replace both:

  ```js
  startKeepalive() {
    chrome.alarms.create(this.KEEPALIVE_NAME, { periodInMinutes: 0.4 });
    this._flushInterval = setInterval(() => this.flushBuffer(), this.FLUSH_INTERVAL);
  },

  stopKeepalive() {
    chrome.alarms.clear(this.KEEPALIVE_NAME);
    if (this._flushInterval) {
      clearInterval(this._flushInterval);
      this._flushInterval = null;
    }
  }
  ```

- [ ] **Step 2: Delete the standalone setInterval at line 358**

  Find and delete this line (it's just after `SW.init();`):
  ```js
  setInterval(() => SW.flushBuffer(), SW.FLUSH_INTERVAL);
  ```
  It is now handled inside `startKeepalive()`.

- [ ] **Step 3: Reload extension and verify**

  Start a recording, stop it. In SW console, verify `[DH] flushBuffer called` stops incrementing shortly after stop (the SW may fire a few more times while winding down, then go quiet).

- [ ] **Step 4: Commit**

  ```bash
  git add background/service-worker.js
  git commit -m "fix: clear setInterval on session stop to prevent perpetual flushBuffer calls"
  ```

---

## Task 6: Fix Suspect C — Bridge.js Idempotency Guard

**Files:**
- Modify: `content/bridge.js`

The current guard uses `chrome.runtime?.id` to detect SW termination, which can fail and allow a second listener to be registered on HMR reloads.

- [ ] **Step 1: Simplify the idempotency guard**

  Open `content/bridge.js`. Replace the current guard block:
  ```js
  if (window.__debugHelperBridge) {
    try { if (chrome.runtime?.id) return; } catch {}
  }
  window.__debugHelperBridge = true;
  ```

  With a simple, reliable check:
  ```js
  if (window.__debugHelperBridge) return;
  window.__debugHelperBridge = true;
  ```

  This prevents any double-registration regardless of SW state. The listener itself handles SW unavailability via `.catch(() => {})`.

  > **Why not also removeEventListener on `recording:stop`?** The spec mentioned that approach, but it is superseded by Task 4: once the capture scripts stop posting messages when not recording, the bridge has nothing to relay. Removing the listener would add complexity (named function, message channel back into ISOLATED world) for no practical benefit given Task 4 already silences the source. If Task 4 has any gap (e.g., page loads before `recording:stop` arrives), the bridge will relay a few extra messages that hit `bufferEvent()` → early-exit on no active session — harmless. The guard fix alone closes the double-registration bug.

- [ ] **Step 2: Reload extension and verify bridge still works**

  Start a recording. Confirm console logs and network events appear in the side panel (bridge is still relaying correctly).

  Trigger several HMR reloads while recording is active. Confirm events still appear (no regression from the guard change).

- [ ] **Step 3: Commit**

  ```bash
  git add content/bridge.js
  git commit -m "fix: simplify bridge.js idempotency guard to prevent HMR double-registration"
  ```

---

## Task 7: Remove Instrumentation

**Files:**
- Modify: `background/service-worker.js` (remove 2 `console.count` lines)
- Modify: `content/bridge.js` (remove 1 `console.count` line)

- [ ] **Step 1: Remove from service-worker.js**

  Remove the two `// TEMP instrumentation` lines added in Task 1 Steps 1 and 2:
  - `console.count('[DH] flushBuffer called');` from `flushBuffer()`
  - `console.count('[DH] bufferEvent: ' + event.type);` from `bufferEvent()`

- [ ] **Step 2: Remove from bridge.js**

  Remove `console.count('[DH] bridge relay: ' + e.data.type);` from the message listener.

- [ ] **Step 3: Verify no `[DH]` logs appear**

  Reload extension. Open SW DevTools console. Run a recording session. Confirm no `[DH]` prefixed counts appear.

- [ ] **Step 4: Commit**

  ```bash
  git add background/service-worker.js content/bridge.js
  git commit -m "chore: remove temporary CPU spike diagnostic instrumentation"
  ```

---

## Task 8: Final Validation

**No code changes — repeat the profiler test from Task 2 to confirm the fix.**

- [ ] **Step 1: Run Phase 1 profiler again**

  Repeat the exact steps from Task 2: open SW DevTools Performance, run the HMR scenario on localhost:3000, start recording, trigger HMR reloads, stop recording, wait 2 minutes.

- [ ] **Step 2: Check Task Manager**

  CPU for the extension row should drop to 0% (or near 0%) within seconds of stopping recording and stay there.

- [ ] **Step 3: Check flame chart**

  After stop, the flame chart should show no significant activity. `flushBuffer`, `bufferEvent`, and `handleMessage` should not appear in the after-stop period.

- [ ] **Step 4: Confirm on a documentation page**

  Navigate to a static documentation page while recording is off. Confirm CPU stays at 0% (no ongoing message traffic from capture scripts).

- [ ] **Step 5: Final commit**

  If everything is clean:
  ```bash
  git log --oneline -6
  # Verify the 4 fix commits are present and clean
  ```
