# AGENT.md — Guitar Chord / Scale Theory Suite (GitHub Pages)

**Repository (GitHub Pages):** https://quantumq1981.github.io/Guitar-Chord-Scale-Theory-Suite/  
**Primary objective:** A single-file (or minimal-file) browser app suite for guitar theory + ear-training + practice utilities (scales, chords, circle of fifths, progressions, quizzes), deployable via GitHub Pages.

---

## 1) Current State Summary (What Works / What’s Blocked)

### Working
- The app loads cleanly on GitHub Pages.
- Console shows **zero errors** (per latest user report).
- UI renders and core non-audio interactions function.

### Blocked / High Priority Bug
- **Audio playback is silent** on the deployed site (particularly iOS Safari / mobile), which makes ear-training quizzes functionally unusable.
- This can occur with **no console errors**, typically due to Web Audio being blocked/suspended until a user gesture, and/or `AudioContext.resume()` not being awaited.

**Status:** Requires patch (see §4 Audio Fix Workflow).

---

## 2) Design Constraints / Principles

- **Web-first:** Must run on GitHub Pages with no backend dependency.
- **Mobile compatibility:** iOS Safari is a primary target and requires explicit audio unlocking.
- **Low friction UX:** Don’t require developer tools; include an “Enable Audio” button and an on-screen audio status.
- **Single source of truth for audio:** All sound output routes through a canonical audio engine to avoid regressions.

---

## 3) Files / Artifacts Mentioned So Far

These were used during development iterations (may or may not be the currently deployed file):

- `/mnt/data/GuitarscalemasterV1.5fullyworking.html`  
- `/mnt/data/guitartheoryappenhanced.html`  
- `/mnt/data/PHASE_3_PROJECT_BRIEF.md`

**Note:** The deployed GitHub Pages file(s) may differ. Always apply fixes to the actual files used by GitHub Pages (commonly `/index.html` and/or `/assets/*.js`).

---

## 4) Audio Fix Workflow (Drop-in Patch + Wiring)

### Problem Pattern
On iOS Safari and some mobile browsers:
- `AudioContext` starts in `suspended`
- `resume()` is async and must be awaited
- audio must be started from a user gesture (touch/click/keydown)
- scheduling oscillators before context is `running` can yield silence with no errors

### Required Fix (Canonical WebAudio Unlock Module)
**Implementation rules:**
1. Add a module that provides `ensureAudioContext()` (async) and `initAudioContext()` (legacy wrapper).
2. Pre-unlock on first `pointerdown/touchstart/mousedown/keydown`.
3. Ensure **every audio entrypoint** awaits the context before creating/scheduling oscillators.

#### Drop-in Module (must be included near top of main script)
```js
const AudioContextClass = window.AudioContext || window.webkitAudioContext;

let audioCtx = null;
let __audioUnlockPromise = null;

async function ensureAudioContext() {
  if (!AudioContextClass) return null;

  if (!audioCtx) {
    audioCtx = new AudioContextClass({ latencyHint: "interactive" });
  }

  if (audioCtx.state !== "running") {
    try { await audioCtx.resume(); } catch (e) {}
  }

  // iOS unlock: play a silent 1-sample buffer once
  if (audioCtx.state === "running" && !audioCtx.__unlocked) {
    const buffer = audioCtx.createBuffer(1, 1, audioCtx.sampleRate);
    const src = audioCtx.createBufferSource();
    src.buffer = buffer;
    src.connect(audioCtx.destination);
    src.start(0);
    audioCtx.__unlocked = true;
  }

  const statusEl = document.getElementById("audioStatus");
  if (statusEl) statusEl.textContent = `Audio: Ready (${audioCtx.state})`;

  return audioCtx;
}

function initAudioContext() {
  if (!__audioUnlockPromise) __audioUnlockPromise = ensureAudioContext();
  return audioCtx;
}

["pointerdown","touchstart","mousedown","keydown"].forEach(evt => {
  document.addEventListener(evt, () => {
    if (!__audioUnlockPromise) __audioUnlockPromise = ensureAudioContext();
  }, { once: true, passive: true });
});
