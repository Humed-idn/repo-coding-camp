# Design Document — Cat Petting Clicker

## Overview

The Cat Petting Clicker is a single-page web application (one `index.html` file) that runs entirely in the browser with no build step. Users click or tap a cute animated SVG cat to accumulate an Affection Counter, trigger randomised visual reactions, and unlock milestone celebrations. The experience centres on immediacy and delight: every tap produces an instant animation, floating emoji particles, and a counter update within 100 ms.

**Core design goals:**

- Zero server, zero build pipeline — HTML + CDN-delivered Tailwind + Vanilla JS.
- All animation via CSS keyframes and CSS custom properties; JavaScript only orchestrates state transitions.
- Persistent state (counter, milestones) via `localStorage`.
- Accessible: keyboard-operable, ARIA-labelled, `prefers-reduced-motion` aware.
- Lighthouse Performance ≥ 90 with no lazy-loading needed (single tiny file).

---

## Architecture

### Single-File Structure

The entire app lives in `index.html`. Sections are logically ordered:

```
index.html
├── <head>
│   ├── Google Fonts (Quicksand, Varela Round) — preconnect + stylesheet link
│   ├── Tailwind CSS CDN script
│   └── <style> — all custom CSS keyframes, utility overrides, glassmorphism tokens
├── <body>  (Tailwind: min-h-screen, gradient bg, flex centering)
│   ├── #milestone-overlay  — full-screen celebration layer (hidden by default)
│   ├── #particle-layer     — absolute-positioned container for all particles
│   ├── #cat-container      — centres the cat; receives click/keydown events
│   │   └── #cat-svg        — the SVG cat character
│   ├── #counter-bubble     — glassmorphism pill showing affection count
│   ├── #happiness-section  — label + meter wrapper
│   │   └── #happiness-bar  — fill element
│   └── <script>            — all application logic (one IIFE)
```

### Module Boundaries (within the single `<script>`)

Although everything is in one file, the JavaScript is structured as distinct logical modules inside an IIFE:

```
(function () {
  // 1. Constants & configuration
  // 2. State object
  // 3. DOM references
  // 4. Storage module      — read/write localStorage
  // 5. ReactionPool module — pool definition + shuffle-window selection
  // 6. StateMachine module — idle / reacting / milestone states
  // 7. ParticleSystem      — spawn, animate, cap, remove
  // 8. UIRenderer          — counter, happiness meter, milestone overlay
  // 9. EventBus            — petAction dispatcher
  // 10. Init               — wire up listeners, load persisted state
})();
```

This separation keeps concerns isolated without requiring a bundler.

### Dependency Graph

```
EventBus ──► StateMachine ──► ReactionPool
                │                  │
                ▼                  ▼
           UIRenderer          ParticleSystem
                │
                ▼
           StorageModule
```

---

## Components and Interfaces

### 1. Cat SVG Character

The cat is a single inline `<svg>` element (~200–300 lines) embedded directly in the HTML. Embedding avoids a network request and allows JavaScript/CSS to target internal SVG elements by ID.

**Anatomy of the SVG:**

| SVG group / element | Purpose |
|---|---|
| `#cat-body` | Main torso ellipse; carries `cat-idle-float` animation |
| `#cat-head` | Head circle atop the body |
| `#cat-ears` | Two triangular ear shapes |
| `#cat-eyes` | Two eye circles (default open); swapped classes per reaction |
| `#cat-mouth` | Small curved path (neutral smile) |
| `#cat-tail` | Curved path; carries `cat-idle-tail-sway` animation |
| `#cat-cheeks` | Two soft circle blushes (hidden by default, shown on blush reactions) |
| `#cat-sparkles` | Small star/sparkle paths (hidden by default, shown on shimmer reaction) |

**Reaction CSS classes applied to `#cat-container`:**

Each reaction class overrides specific SVG sub-element appearances via CSS child selectors:

| Class | Visual effect on cat |
|---|---|
| `reaction-happy-squint` | Eyes become curved arcs (happy squint paths shown) |
| `reaction-blush` | Cheeks become visible, soft pink circles |
| `reaction-headbutt` | Whole cat translates forward then recoils (`headbutt` keyframe) |
| `reaction-kneading` | Paws animate up-down alternating (`kneading` keyframe on paw paths) |
| `reaction-slow-blink` | Eye opacity cycles 1→0→1 over 600 ms |
| `reaction-spin` | Entire `#cat-container` rotates 360° via `spin` keyframe |
| `reaction-startle-bounce` | Cat scales up then back with a quick jitter (`startle` keyframe) |
| `reaction-purr-shimmer` | Sparkles become visible, cat pulses scale 1→1.05→1 |

### 2. Animation System

All animations are defined as CSS `@keyframes` inside the `<style>` block.

**Idle animations (always playing when not reacting):**

```css
@keyframes cat-idle-float {
  0%, 100% { transform: translateY(0px); }
  50%       { transform: translateY(-8px); }
}
/* Applied to #cat-body, duration: 2.8s ease-in-out infinite */

@keyframes cat-idle-tail-sway {
  0%, 100% { transform: rotate(0deg); transform-origin: bottom left; }
  50%       { transform: rotate(12deg); }
}
/* Applied to #cat-tail, duration: 3.2s ease-in-out infinite */
```

Cycle duration is fixed at session start to the same variant (2.8 s float, 3.2 s tail) so the requirement of "same variant for duration of session" is satisfied inherently.

**Reaction keyframes (selected by CSS class added to `#cat-container`):**

```css
@keyframes headbutt {
  0%   { transform: translateX(0); }
  30%  { transform: translateX(18px); }
  60%  { transform: translateX(-6px); }
  100% { transform: translateX(0); }
}

@keyframes kneading {
  0%, 100% { transform: translateY(0); }
  50%      { transform: translateY(-6px); }
}
/* applied to paw path elements with alternating animation-delay */

@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

@keyframes startle {
  0%   { transform: scale(1); }
  20%  { transform: scale(1.25) translateY(-10px); }
  40%  { transform: scale(0.9) translateX(4px); }
  60%  { transform: scale(1.1) translateX(-4px); }
  100% { transform: scale(1); }
}

@keyframes purr-pulse {
  0%, 100% { transform: scale(1); }
  50%      { transform: scale(1.04); }
}
```

**Particle keyframes:**

Each particle receives an inline `--dx` and `--dy` CSS custom property set by JavaScript at spawn time, allowing a single keyframe definition to produce varied arcs:

```css
@keyframes particle-float {
  0%   { transform: translate(0, 0) scale(1);   opacity: 1; }
  70%  { opacity: 1; }
  100% { transform: translate(var(--dx), var(--dy)) scale(0.6); opacity: 0; }
}
```

**`prefers-reduced-motion` override:**

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.001ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 150ms !important;
  }
  .particle { display: none !important; }
}
```

This disables particle display and collapses all animations to a near-instant opacity fade, satisfying Requirement 9.3.

### 3. State Machine

The app has three top-level states:

```
         petAction
  ┌──────────────────────────────┐
  │                              ▼
IDLE ──── petAction ────► REACTING ──── reactionComplete ────► IDLE
                               │
                               │  petAction (while reacting)
                               ▼
                        QUEUE (up to 100)
                               │
                   last queue item completes
                               ▼
                             IDLE

MILESTONE (overlay) fires on top of REACTING/IDLE,
then resolves back to normal flow after 2000 ms.
```

**State object:**

```js
const state = {
  mode: 'idle',          // 'idle' | 'reacting' | 'milestone'
  queue: [],             // PetAction objects, max length 100
  activeReaction: null,  // current Reaction descriptor or null
  affectionCount: 0,     // non-negative integer ≤ 999,999
  milestonesReached: new Set(), // Set<number>
};
```

**Transitions:**

| Event | From state | Action | To state |
|---|---|---|---|
| `petAction` | `idle` | Dequeue next reaction (or use this one), begin animation | `reacting` |
| `petAction` | `reacting` | Push to queue if `queue.length < 100`; if queue full, silently drop | `reacting` |
| `reactionComplete` | `reacting` | Check queue; if non-empty, begin next; else go idle | `idle` or `reacting` |
| `milestoneHit` | any | Pause normal input, show overlay | `milestone` |
| `milestoneComplete` | `milestone` | Restore normal input | previous state |

**Implementation note on Requirement 1.5 (at most one reaction at a time):** The `animationend` DOM event on the cat container fires `reactionComplete`. The state machine ensures `beginReaction()` is only called when `state.mode !== 'reacting'` or when transitioning from queue processing.

### 4. Reaction Pool Module

```js
const REACTION_POOL = [
  { id: 'happy-squint',    cssClass: 'reaction-happy-squint',    duration: 1200, particles: ['♥','💕','😸'] },
  { id: 'blush',           cssClass: 'reaction-blush',           duration: 1000, particles: ['🌸','💗','✿'] },
  { id: 'headbutt',        cssClass: 'reaction-headbutt',        duration: 900,  particles: ['💥','⭐','✨'] },
  { id: 'kneading',        cssClass: 'reaction-kneading',        duration: 2000, particles: ['🐾','💕','🐱'] },
  { id: 'slow-blink',      cssClass: 'reaction-slow-blink',      duration: 1400, particles: ['💤','😌','🌙'] },
  { id: 'spin',            cssClass: 'reaction-spin',            duration: 800,  particles: ['💫','✨','⭐'] },
  { id: 'startle-bounce',  cssClass: 'reaction-startle-bounce',  duration: 950,  particles: ['❗','😱','💦'] },
  { id: 'purr-shimmer',    cssClass: 'reaction-purr-shimmer',    duration: 1600, particles: ['✨','💛','🌟'] },
];
```

**Shuffle-window algorithm (Requirement 2.5):**

A sliding window of the last 5 reactions is maintained. When selecting the next reaction, the pool is filtered to exclude any reaction that has appeared ≥ 2 times in the window. A random item is picked from the filtered subset.

```js
function pickReaction() {
  const window = state.reactionHistory.slice(-5);
  const counts = {};
  window.forEach(id => { counts[id] = (counts[id] || 0) + 1; });
  const eligible = REACTION_POOL.filter(r => (counts[r.id] || 0) < 2);
  const pool = eligible.length > 0 ? eligible : REACTION_POOL;
  return pool[Math.floor(Math.random() * pool.length)];
}
```

`state.reactionHistory` is an array capped at the last 5 entries (shift oldest when length exceeds 5).

### 5. Particle System

**Spawn function:**

```js
function spawnParticles(x, y, emojiSet) {
  const count = 3 + Math.floor(Math.random() * 5); // 3–7
  // Enforce cap
  while (activeparticles.length >= 30) {
    const oldest = activeParticles.shift();
    oldest.remove();
  }
  for (let i = 0; i < count; i++) {
    const el = document.createElement('span');
    el.className = 'particle';
    el.textContent = emojiSet[Math.floor(Math.random() * emojiSet.length)];
    const dx = (Math.random() * 100 - 50) + 'px';   // −50 to +50 px lateral
    const dy = -(40 + Math.random() * 80) + 'px';    // 40–120 px upward (negative Y)
    const dur = 600 + Math.random() * 400;            // 600–1000 ms
    el.style.cssText = `
      position: fixed;
      left: ${x}px; top: ${y}px;
      --dx: ${dx}; --dy: ${dy};
      font-size: 1.4rem;
      pointer-events: none;
      animation: particle-float ${dur}ms ease-out forwards;
    `;
    particleLayer.appendChild(el);
    activeParticles.push(el);
    el.addEventListener('animationend', () => {
      el.remove();
      activeParticles.splice(activeParticles.indexOf(el), 1);
    });
  }
}
```

Particles are positioned via `position: fixed` at the raw viewport coordinates of the click/tap event, satisfying Requirement 8.1. They drift upward (negative Y delta) with lateral spread, satisfying Requirement 8.2. The `animationend` handler removes the DOM element (Requirement 8.3). The cap loop enforces the ≤ 30 limit by removing oldest first (Requirement 8.4). The keyframe fades opacity from 1 to 0 over the last 30% of duration (Requirement 8.5).

### 6. Affection Counter and UI Renderer

**Counter bubble HTML:**

```html
<div id="counter-bubble"
     class="bg-white/30 backdrop-blur-lg rounded-full border border-white/40
            px-8 py-4 flex flex-col items-center gap-1">
  <span class="text-sm font-semibold text-pink-500 tracking-widest uppercase">Pets</span>
  <span id="counter-value" class="text-4xl font-bold text-pink-700">0</span>
</div>
```

**Counter update (scale-bounce):**

```js
function updateCounter(newVal) {
  state.affectionCount = newVal;
  counterValueEl.textContent = newVal.toLocaleString();
  counterValueEl.classList.remove('counter-bump');
  void counterValueEl.offsetWidth; // force reflow to restart animation
  counterValueEl.classList.add('counter-bump');
}
```

```css
@keyframes counter-bump {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.3); }
  100% { transform: scale(1); }
}
.counter-bump { animation: counter-bump 300ms ease-in-out; }
```

**Happiness Meter update:**

```js
function updateHappinessMeter(n) {
  const pct = Math.min(100, Math.floor(Math.log(n + 1) / Math.log(501) * 100));
  happinessBarEl.style.width = pct + '%';
}
```

The CSS transition on `#happiness-bar` is set to `transition: width 400ms ease-in-out` (Requirement 5.3). Overflow is prevented because `Math.min(100, ...)` clamps the value (Requirement 5.4).

### 7. Milestone System

**Milestone definitions:**

```js
const MILESTONES = [10, 50, 100, 250, 500];
```

**Check after each pet action:**

```js
function checkMilestones(newCount, oldCount) {
  for (const m of MILESTONES) {
    if (!state.milestonesReached.has(m) && newCount >= m && oldCount < m ||
        !state.milestonesReached.has(m) && newCount > m) {
      // handles jump-past case (Req 6.5)
      triggerMilestone(m);
    }
  }
}
```

The `newCount > m` branch with the `!has(m)` guard catches cases where the counter jumps past a milestone threshold (Requirement 6.5).

**Milestone celebration:**

```js
function triggerMilestone(value) {
  state.milestonesReached.add(value);
  saveStorage();
  state.mode = 'milestone';
  showMilestoneOverlay(value);  // spawns ≥20 particles across full viewport, shows text
  setTimeout(() => {
    hideMilestoneOverlay();
    state.mode = 'idle';
  }, 2000);
}
```

Full-screen particles are spawned at random positions within `window.innerWidth` × `window.innerHeight` coordinates, with at least 20 particles (Requirement 6.2). The overlay text remains visible for the 2000 ms duration (Requirement 6.3).

### 8. Storage Module

```js
const Storage = {
  load() {
    const rawCount = localStorage.getItem('catPettingCount');
    const rawMilestones = localStorage.getItem('catMilestonesReached');
    const count = parseInt(rawCount, 10);
    state.affectionCount = (Number.isInteger(count) && count >= 0) ? count : 0;
    try {
      const parsed = JSON.parse(rawMilestones);
      state.milestonesReached = new Set(Array.isArray(parsed) ? parsed : []);
    } catch {
      state.milestonesReached = new Set();
    }
  },
  save() {
    localStorage.setItem('catPettingCount', state.affectionCount);
    localStorage.setItem('catMilestonesReached',
      JSON.stringify([...state.milestonesReached]));
  },
};
```

The invalid-value guard (`Number.isInteger(count) && count >= 0`) covers Requirement 4.5.

---

## Data Models

### Runtime State Object

```ts
interface AppState {
  mode: 'idle' | 'reacting' | 'milestone';
  queue: PetAction[];           // max 100 items
  reactionHistory: string[];    // last 5 reaction IDs for shuffle window
  activeReaction: Reaction | null;
  affectionCount: number;       // 0 ≤ n ≤ 999_999
  milestonesReached: Set<number>;
  activeParticles: HTMLElement[];  // max 30 items
}

interface PetAction {
  x: number;   // viewport X of click/tap
  y: number;   // viewport Y of click/tap
}

interface Reaction {
  id: string;
  cssClass: string;   // applied to #cat-container
  duration: number;   // ms; how long before reactionComplete fires
  particles: string[]; // emoji array
}
```

### localStorage Schema

| Key | Type | Description |
|---|---|---|
| `catPettingCount` | string (integer) | Serialised affection counter |
| `catMilestonesReached` | string (JSON array of numbers) | Milestone values already fired |

### Reaction Pool Record

```ts
interface ReactionRecord {
  id: string;        // kebab-case unique identifier
  cssClass: string;  // CSS class toggled on #cat-container
  duration: number;  // animation duration in ms (800–3000)
  particles: string[]; // 3-item emoji array for particle selection
}
```

### Particle Element (runtime only)

Particles are ephemeral DOM nodes (`<span class="particle">`); they carry their state entirely in inline CSS custom properties (`--dx`, `--dy`) and are removed from the DOM on `animationend`. No persistent data model is needed.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Affection Counter persistence round-trip

*For any* non-negative integer affection count saved to `localStorage` under `catPettingCount`, reading it back and initialising the counter should produce the same integer value.

**Validates: Requirements 4.3, 4.4**

---

### Property 2: Invalid localStorage value defaults to zero

*For any* string stored under `catPettingCount` that is not a non-negative integer (including `null`, negative numbers, floats, and arbitrary strings), the counter initialisation should always produce 0.

**Validates: Requirements 4.5**

---

### Property 3: Happiness meter formula is monotonically non-decreasing and clamped

*For any* two non-negative integers `a` and `b` where `a ≤ b`, the happiness percentage computed for `a` shall be less than or equal to the happiness percentage computed for `b`, and for all `n ≥ 500`, the percentage shall equal 100.

**Validates: Requirements 5.2, 5.4**

---

### Property 4: Reaction queue never exceeds capacity

*For any* sequence of pet actions delivered while the state machine is in the `reacting` state, the length of the reaction queue shall never exceed 100, and no action delivered beyond the 100-item limit shall cause the queue to grow further.

**Validates: Requirements 1.4**

---

### Property 5: Shuffle window prevents over-repetition

*For any* sequence of 5 or more consecutive pet actions, no single reaction ID shall appear more than twice within any sliding window of 5 consecutive reactions drawn from the pool selection algorithm.

**Validates: Requirements 2.5**

---

### Property 6: Particle count cap is maintained

*For any* sequence of pet actions that would spawn an arbitrary number of particles, the number of simultaneously active DOM particle elements shall never exceed 30, with oldest particles removed first when the cap would be exceeded.

**Validates: Requirements 8.4**

---

### Property 7: Milestone fires for every crossed threshold

*For any* pair of consecutive counter values `(oldCount, newCount)` where `newCount > oldCount`, every milestone threshold `m` in `[10, 50, 100, 250, 500]` that satisfies `oldCount < m ≤ newCount` and has not been previously triggered shall have its celebration fired exactly once.

**Validates: Requirements 6.1, 6.5**

---

### Property 8: Milestone celebration fires at most once per threshold

*For any* milestone threshold that has already been triggered and stored in `localStorage`, subsequent pet actions that reach or exceed that threshold shall not trigger a second celebration for that threshold.

**Validates: Requirements 6.4**

---

### Property 9: Milestone persistence round-trip

*For any* set of triggered milestone values, serialising them to `localStorage` under `catMilestonesReached` and then deserialising them on the next load should produce a set containing exactly the same values.

**Validates: Requirements 6.4**

---

### Property 10: Particle arc stays within spec bounds

*For any* spawned particle, the absolute vertical displacement shall be between 40 px and 120 px upward, and the absolute lateral displacement shall be between 0 px and 50 px in either direction.

**Validates: Requirements 8.2**

---

## Error Handling

### localStorage Unavailability

`localStorage` may be unavailable (private browsing with storage blocked, quota exceeded, or storage disabled). All `localStorage` reads/writes are wrapped in `try/catch`:

```js
function safeSetItem(key, value) {
  try { localStorage.setItem(key, value); }
  catch { /* silent — app continues with in-memory state only */ }
}
function safeGetItem(key) {
  try { return localStorage.getItem(key); }
  catch { return null; }
}
```

The app degrades gracefully: state persists in memory for the session but is not saved across reloads.

### Google Fonts Load Failure

Requirement 7.2 mandates a system font fallback. The font stack on `body` is:

```css
font-family: 'Quicksand', 'Varela Round', -apple-system, BlinkMacSystemFont,
             'Segoe UI', sans-serif;
```

No JavaScript font-load detection is needed; the browser falls back automatically.

### Animation Event Reliability

`animationend` may fire multiple times if several CSS animations run simultaneously on the same element (e.g., when a reaction class adds animations to multiple sub-elements). The `reactionComplete` handler uses a **one-shot guard**:

```js
catContainer.addEventListener('animationend', (e) => {
  // Only process the animation on the container element itself,
  // not bubbled events from child SVG elements.
  if (e.target !== catContainer) return;
  if (state.mode !== 'reacting') return;
  endReaction();
}, false);
```

If the reaction CSS class is on `#cat-container` directly, and the keyframe is defined on that element, this guard ensures exactly one `endReaction()` call per reaction.

As a safety net, each `beginReaction()` also sets a `setTimeout` fallback:

```js
function beginReaction(reaction, petAction) {
  // ...apply class, spawn particles...
  state.reactionTimeout = setTimeout(endReaction, reaction.duration + 200);
}
function endReaction() {
  clearTimeout(state.reactionTimeout);
  // ...remove class, process queue...
}
```

This ensures the state machine never gets stuck if `animationend` fails to fire.

### Affection Counter Maximum

The counter is clamped at 999,999 before saving:

```js
state.affectionCount = Math.min(999_999, state.affectionCount + 1);
```

This satisfies Requirement 1.2.

### Invalid Pet Actions (zero-size touches)

Touch events with `changedTouches.length === 0` are ignored silently.

---

## Testing Strategy

### Approach

Because this is a zero-build, vanilla JS, single-file app, testing is split between:

1. **Pure logic unit tests** — extracted functions tested in Node (or a browser test harness like Jasmine CDN or `QUnit`).
2. **Property-based tests** — for the stateless logic functions using a PBT library.
3. **Integration/manual tests** — browser-based, covering visual output, animations, and localStorage.

### Property-Based Testing

**Library:** [fast-check](https://github.com/dubzzz/fast-check) (browser-compatible UMD build via CDN, or run in Node).

**Minimum iterations per property:** 100.

**Tag format in test code:**
```js
// Feature: cat-petting-clicker, Property N: <property text>
```

**Properties to implement as PBT tests:**

| Property | Test description | fast-check arbitraries |
|---|---|---|
| Property 1 | Counter persistence round-trip | `fc.integer({ min: 0, max: 999_999 })` |
| Property 2 | Invalid localStorage defaults to 0 | `fc.oneof(fc.string(), fc.float(), fc.constant(null), fc.integer({ max: -1 }))` |
| Property 3 | Happiness formula monotone + clamped | `fc.tuple(fc.nat(), fc.nat()).map(([a,b]) => [Math.min(a,b), Math.max(a,b)])` |
| Property 4 | Queue never exceeds 100 | `fc.array(fc.record({x: fc.float(), y: fc.float()}), { maxLength: 500 })` |
| Property 5 | Shuffle window ≤ 2 repeats in 5 | `fc.integer({ min: 5, max: 200 })` (number of picks to simulate) |
| Property 6 | Particle cap ≤ 30 | `fc.array(fc.record({x: fc.float(), y: fc.float()}), { maxLength: 200 })` |
| Property 7 | Milestone fires for every crossed threshold | `fc.tuple(fc.nat({ max: 499 }), fc.nat({ max: 500 }).map(d => d + 1))` |
| Property 8 | Milestone fires at most once | `fc.integer({ min: 0, max: 600 })` (counter values that repeatedly cross a milestone) |
| Property 9 | Milestone persistence round-trip | `fc.subarray([10, 50, 100, 250, 500])` |
| Property 10 | Particle arc bounds | `fc.record({x: fc.float({min:0,max:1920}), y: fc.float({min:0,max:1080})})` |

**Test configuration:**

```js
fc.configureGlobal({ numRuns: 100 });
```

### Unit Tests (Example-Based)

Focused on specific scenarios not covered by properties:

- Counter display formats large numbers with `toLocaleString` (e.g. `12345` → `"12,345"`).
- Happiness meter at `n = 0` returns 0%, at `n = 500` returns 100%.
- `pickReaction()` always returns a member of `REACTION_POOL`.
- Reaction class is removed from `#cat-container` after `endReaction()`.
- Queue size stays at 100 when 101 actions are delivered rapidly.
- Milestone overlay text is visible for ≥ 3000 ms (timer mock).

### Manual / Integration Tests

Browser-based checks that require visual or DOM inspection:

- Idle animation plays on page load; tail sways and body floats.
- Clicking the cat triggers a visible particle burst at cursor coordinates.
- Affection counter updates within visible frame after click.
- Happiness bar transitions smoothly on counter change.
- Milestone overlay appears at counts 10, 50, 100, 250, 500.
- `prefers-reduced-motion: reduce` suppresses particles and bounce.
- Keyboard: `Tab` to cat, `Enter`/`Space` triggers reaction.
- Focus indicator is visible with ≥ 2 px outline and ≥ 3:1 contrast.
- Page reloads restore counter and skip already-triggered milestones.
- Lighthouse CLI: Performance ≥ 90 on desktop simulation.

### Accessibility Testing

- Run [axe-core](https://github.com/dequelabs/axe-core) in browser devtools to catch ARIA issues.
- Manually navigate with NVDA or VoiceOver to confirm "Pet the cat" announcement.
- Verify `role="button"` and `tabindex="0"` on the cat SVG container.
- Note: full WCAG validation requires expert accessibility review and assistive technology testing beyond automated tooling.
