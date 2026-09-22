# Implementation Plan: Cat Petting Clicker

## Overview

Build a single `index.html` file containing all HTML structure, inline CSS (Tailwind CDN + custom keyframes), and a Vanilla JavaScript IIFE that implements the full Cat Petting Clicker experience — idle animations, reaction pool, particle system, affection counter, happiness meter, milestone celebrations, accessibility, and property-based tests.

---

## Tasks

- [x] 1. Project scaffolding
  - [x] 1.1 Create `index.html` with HTML5 boilerplate, `<meta>` viewport tag, and document title "Cat Petting Clicker"
    - Add `<link>` preconnect + stylesheet for Google Fonts (Quicksand, Varela Round)
    - Add Tailwind CSS CDN `<script>` tag
    - Set `<body>` classes: `min-h-screen flex items-center justify-center` with the gradient background (`bg-gradient-to-br from-[#FFDEE9] to-[#B5FFFC]`)
    - Define CSS custom properties and `font-family` stack with system sans-serif fallback
    - Add empty `<style>` block (to be filled in later tasks) and empty `<script>` block
    - _Requirements: 7.1, 7.2_

  - [x] 1.2 Add DOM skeleton inside `<body>`: `#milestone-overlay`, `#particle-layer`, `#cat-container` (with `#cat-svg` placeholder), `#counter-bubble` with `#counter-value`, `#happiness-section` with `#happiness-bar`, and a counter reset button
    - Apply glassmorphism Tailwind classes (`bg-white/30 backdrop-blur-lg rounded-full border border-white/40`) to counter bubble and happiness meter wrapper
    - _Requirements: 7.3, 4.1, 5.1_

---

- [x] 2. Cat SVG character
  - [x] 2.1 Implement the inline `<svg>` cat inside `#cat-svg` with all named groups and elements: `#cat-body`, `#cat-head`, `#cat-ears`, `#cat-eyes`, `#cat-mouth`, `#cat-tail`, `#cat-cheeks`, `#cat-sparkles`
    - Use a soft pastel colour palette; cheeks and sparkles default to `display: none`
    - Apply `aspect-square` utility and a drop shadow / glow with blur ≥ 8px, opacity 20–60%
    - _Requirements: 7.4_

---

- [x] 3. Idle animations (CSS keyframes)
  - [x] 3.1 Add `@keyframes cat-idle-float` (0%/100% translateY(0), 50% translateY(−8px)) applied to `#cat-body` with `animation: cat-idle-float 2.8s ease-in-out infinite`
    - Add `@keyframes cat-idle-tail-sway` (0%/100% rotate(0deg), 50% rotate(12deg), transform-origin bottom left) applied to `#cat-tail` with `animation: cat-idle-tail-sway 3.2s ease-in-out infinite`
    - _Requirements: 3.1_

---

- [x] 4. IIFE script structure and state object
  - [x] 4.1 Write the outer `(function () { ... })()` IIFE in the `<script>` block with clearly labelled sections: Constants & configuration, State object, DOM references, and stubs for each module (Storage, ReactionPool, StateMachine, ParticleSystem, UIRenderer, EventBus, Init)
    - Define the `state` object: `{ mode: 'idle', queue: [], reactionHistory: [], activeReaction: null, affectionCount: 0, milestonesReached: new Set(), activeParticles: [], reactionTimeout: null }`
    - Capture all DOM references (`catContainer`, `counterValueEl`, `happinessBarEl`, `particleLayer`, `milestoneOverlay`)
    - _Requirements: 1.1, 1.4_

---

- [ ] 5. Storage module
  - [-] 5.1 Implement `Storage.load()` inside the IIFE:
    - Wrap all `localStorage` calls in `try/catch` via `safeGetItem` / `safeSetItem` helpers
    - Parse `catPettingCount`: use `parseInt(raw, 10)`; set `state.affectionCount` to result only if `Number.isInteger(result) && result >= 0`, otherwise 0
    - Parse `catMilestonesReached`: `JSON.parse`; accept only arrays, fall back to empty `Set`
    - _Requirements: 4.3, 4.4, 4.5, 6.4_

  - [-] 5.2 Implement `Storage.save()`:
    - Write `state.affectionCount` to `catPettingCount`
    - Write `JSON.stringify([...state.milestonesReached])` to `catMilestonesReached`
    - _Requirements: 4.3, 6.4_

  - [ ]* 5.3 Write property test for counter persistence round-trip (Property 1)
    - **Property 1: Affection Counter persistence round-trip**
    - Arbitrary: `fc.integer({ min: 0, max: 999_999 })`
    - Simulate `Storage.save()` then `Storage.load()` on the same value; assert `state.affectionCount` equals the original integer
    - **Validates: Requirements 4.3, 4.4**

  - [ ]* 5.4 Write property test for invalid localStorage defaults to zero (Property 2)
    - **Property 2: Invalid localStorage value defaults to zero**
    - Arbitrary: `fc.oneof(fc.string(), fc.float(), fc.constant(null), fc.integer({ max: -1 }))`
    - Stub `localStorage.getItem('catPettingCount')` with each generated value; assert `state.affectionCount === 0` after `Storage.load()`
    - **Validates: Requirements 4.5**

  - [ ]* 5.5 Write property test for milestone persistence round-trip (Property 9)
    - **Property 9: Milestone persistence round-trip**
    - Arbitrary: `fc.subarray([10, 50, 100, 250, 500])`
    - Serialise to `catMilestonesReached`, deserialise, assert the resulting `Set` contains exactly the same values
    - **Validates: Requirements 6.4**

---

- [ ] 6. Reaction Pool module
  - [-] 6.1 Define `REACTION_POOL` array with all 8 reaction objects: `happy-squint`, `blush`, `headbutt`, `kneading`, `slow-blink`, `spin`, `startle-bounce`, `purr-shimmer` — each with `id`, `cssClass`, `duration` (800–3000 ms), and `particles` emoji array
    - _Requirements: 2.1, 2.2_

  - [~] 6.2 Implement `pickReaction()` with the shuffle-window algorithm:
    - Slice last 5 entries from `state.reactionHistory`; count occurrences per `id`
    - Filter `REACTION_POOL` to reactions with count < 2; fall back to full pool if filter yields empty
    - Pick uniformly at random; push selected `id` to `state.reactionHistory`, trimming to last 5
    - _Requirements: 2.5_

  - [ ]* 6.3 Write property test for shuffle-window ≤ 2 repeats in 5 (Property 5)
    - **Property 5: Shuffle window prevents over-repetition**
    - Arbitrary: `fc.integer({ min: 5, max: 200 })`
    - Simulate N calls to `pickReaction()` with a clean `reactionHistory`; for every sliding window of 5 consecutive picks, assert no single reaction ID appears more than twice
    - **Validates: Requirements 2.5**

---

- [ ] 7. CSS reaction classes and keyframes
  - [-] 7.1 Add all reaction `@keyframes` blocks to the `<style>` tag: `headbutt`, `kneading`, `spin`, `startle`, `purr-pulse`, `slow-blink` (opacity cycle), `reaction-blush` (cheeks visible), `reaction-happy-squint` (eye arcs), `reaction-purr-shimmer` (sparkles + pulse)
    - Ensure each reaction CSS class targets the correct SVG sub-elements via child selectors
    - Add `@keyframes counter-bump` and `.counter-bump` class (scale 1→1.3→1, 300ms)
    - _Requirements: 2.2, 4.2_

---

- [ ] 8. Particle system
  - [~] 8.1 Implement `spawnParticles(x, y, emojiSet)`:
    - Count = `3 + Math.floor(Math.random() * 5)` (3–7)
    - Enforce cap: while `state.activeParticles.length >= 30`, remove and call `.remove()` on oldest
    - Create `<span class="particle">`, set `position: fixed`, `left: x`, `top: y`, random `--dx` (±50 px) and `--dy` (−40 to −120 px) custom properties, random duration (600–1000 ms)
    - Push to `state.activeParticles`; on `animationend` remove element from DOM and splice from array
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

  - [~] 8.2 Add `@keyframes particle-float` to `<style>`: `0%` translate(0,0) scale(1) opacity 1; `70%` opacity 1; `100%` translate(var(--dx), var(--dy)) scale(0.6) opacity 0
    - _Requirements: 8.5_

  - [ ]* 8.3 Write property test for particle count cap ≤ 30 (Property 6)
    - **Property 6: Particle count cap is maintained**
    - Arbitrary: `fc.array(fc.record({ x: fc.float(), y: fc.float() }), { maxLength: 200 })`
    - Call `spawnParticles` for each action using a mock DOM; after each call assert `state.activeParticles.length <= 30`
    - **Validates: Requirements 8.4**

  - [ ]* 8.4 Write property test for particle arc bounds (Property 10)
    - **Property 10: Particle arc stays within spec bounds**
    - Arbitrary: `fc.record({ x: fc.float({ min: 0, max: 1920 }), y: fc.float({ min: 0, max: 1080 }) })`
    - Inspect generated `--dy` values to assert vertical displacement is between 40px and 120px upward, and `--dx` absolute value is between 0px and 50px
    - **Validates: Requirements 8.2**

---

- [ ] 9. Reaction state machine
  - [~] 9.1 Implement `beginReaction(reaction, petAction)`:
    - Add `reaction.cssClass` to `#cat-container`; call `spawnParticles`
    - Set `state.mode = 'reacting'`; set `state.activeReaction = reaction`
    - Set `state.reactionTimeout = setTimeout(endReaction, reaction.duration + 200)` as fallback
    - _Requirements: 1.3, 2.2, 3.2_

  - [~] 9.2 Implement `endReaction()`:
    - `clearTimeout(state.reactionTimeout)`
    - Remove `reaction.cssClass` from `#cat-container`; resume idle (restore animation-play-state)
    - If `state.queue` is non-empty, shift next `petAction` from queue and call `beginReaction` with `pickReaction()` and the dequeued action
    - Otherwise set `state.mode = 'idle'`
    - _Requirements: 2.4, 3.3, 3.4_

  - [~] 9.3 Add `animationend` listener on `#cat-container` with one-shot guard: skip if `e.target !== catContainer` or `state.mode !== 'reacting'`; call `endReaction()`
    - _Requirements: 1.5_

  - [ ]* 9.4 Write property test for reaction queue never exceeds 100 (Property 4)
    - **Property 4: Reaction queue never exceeds capacity**
    - Arbitrary: `fc.array(fc.record({ x: fc.float(), y: fc.float() }), { maxLength: 500 })`
    - Force `state.mode = 'reacting'`; dispatch each action through the queue push path; assert `state.queue.length <= 100` at every step
    - **Validates: Requirements 1.4**

---

- [~] 10. Checkpoint — core loop complete
  - Ensure idle animation plays, clicking the cat logs a reaction class to `#cat-container`, particles appear, and the queue empties after reactions complete. Ask the user if questions arise.

---

- [ ] 11. Affection counter UI
  - [~] 11.1 Implement `updateCounter(newVal)`:
    - Clamp: `newVal = Math.min(999_999, newVal)`
    - Set `state.affectionCount = newVal`; `counterValueEl.textContent = newVal.toLocaleString()`
    - Remove `.counter-bump`, force reflow via `void counterValueEl.offsetWidth`, re-add `.counter-bump`
    - Call `Storage.save()`
    - _Requirements: 4.1, 4.2, 1.2_

---

- [ ] 12. Happiness meter UI
  - [~] 12.1 Implement `updateHappinessMeter(n)`:
    - `const pct = Math.min(100, Math.floor(Math.log(n + 1) / Math.log(501) * 100))`
    - Set `happinessBarEl.style.width = pct + '%'`
    - Ensure `#happiness-bar` has `transition: width 400ms ease-in-out` in CSS
    - _Requirements: 5.2, 5.3, 5.4_

  - [ ]* 12.2 Write property test for happiness formula monotone and clamped (Property 3)
    - **Property 3: Happiness meter formula is monotonically non-decreasing and clamped**
    - Arbitrary: `fc.tuple(fc.nat(), fc.nat()).map(([a, b]) => [Math.min(a,b), Math.max(a,b)])`
    - Assert `happinessPct(a) <= happinessPct(b)` for all `a <= b`, and `happinessPct(n) === 100` for all `n >= 500`
    - **Validates: Requirements 5.2, 5.4**

---

- [ ] 13. Milestone system
  - [~] 13.1 Define `MILESTONES = [10, 50, 100, 250, 500]` and implement `checkMilestones(newCount, oldCount)`:
    - For each `m` in `MILESTONES`: if `!state.milestonesReached.has(m) && newCount >= m` (covers both exact hit and jump-past), call `triggerMilestone(m)`
    - _Requirements: 6.1, 6.5_

  - [~] 13.2 Implement `triggerMilestone(value)`:
    - Add `value` to `state.milestonesReached`; call `Storage.save()`
    - Set `state.mode = 'milestone'`; show `#milestone-overlay` with celebratory text
    - Spawn ≥ 20 particles at random positions within `window.innerWidth × window.innerHeight`
    - `setTimeout(() => { hideMilestoneOverlay(); state.mode = 'idle'; }, 2000)`
    - _Requirements: 6.2, 6.3, 6.4_

  - [ ]* 13.3 Write property test for milestone fires for every crossed threshold (Property 7)
    - **Property 7: Milestone fires for every crossed threshold**
    - Arbitrary: `fc.tuple(fc.nat({ max: 499 }), fc.nat({ max: 500 }).map(d => d + 1))`
    - For each `(oldCount, delta)` pair, compute `newCount = oldCount + delta`; run `checkMilestones`; assert every `m` in `[10,50,100,250,500]` with `oldCount < m <= newCount` was triggered exactly once
    - **Validates: Requirements 6.1, 6.5**

  - [ ]* 13.4 Write property test for milestone fires at most once (Property 8)
    - **Property 8: Milestone celebration fires at most once per threshold**
    - Arbitrary: `fc.integer({ min: 0, max: 600 })`
    - Repeatedly call `checkMilestones` with values that cross the same milestone; assert each milestone's trigger count is ≤ 1
    - **Validates: Requirements 6.4**

---

- [ ] 14. Event wiring
  - [~] 14.1 Implement the `petAction(x, y)` dispatcher:
    - If `state.mode === 'milestone'`, ignore
    - Increment and clamp counter; call `updateCounter` and `updateHappinessMeter`; call `checkMilestones(newCount, oldCount)`
    - If `state.mode === 'idle'`, call `beginReaction(pickReaction(), { x, y })`
    - If `state.mode === 'reacting'` and `state.queue.length < 100`, push `{ x, y }` to `state.queue`; otherwise silently drop
    - _Requirements: 1.1, 1.2, 1.4, 1.5_

  - [~] 14.2 Attach event listeners in `Init`:
    - `click` on `#cat-container`: extract `e.clientX`, `e.clientY`; call `petAction`
    - `touchstart` on `#cat-container`: use `e.changedTouches[0]`; guard against zero-length `changedTouches`; call `petAction`
    - `keydown` on `#cat-container`: if `e.key === 'Enter' || e.key === ' '`, call `petAction` with centre coordinates of the element
    - Set `role="button"`, `tabindex="0"`, `aria-label="Pet the cat"` on `#cat-container`
    - _Requirements: 1.1, 9.1, 9.2_

  - [~] 14.3 Wire reset button: on `click`, set `state.affectionCount = 0`, clear milestones from state and localStorage, call `updateCounter(0)` and `updateHappinessMeter(0)`
    - _Requirements: 4.1_

---

- [ ] 15. Accessibility and reduced-motion
  - [~] 15.1 Add `@media (prefers-reduced-motion: reduce)` block in `<style>`:
    - Collapse all animation/transition durations to `0.001ms` with `!important`
    - Set `animation-iteration-count: 1 !important`; set `transition-duration: 150ms !important`
    - Add `.particle { display: none !important; }`
    - _Requirements: 9.3_

  - [~] 15.2 Add focus indicator CSS for all interactive elements (`#cat-container`, reset button):
    - `:focus-visible { outline: 2px solid #D6336C; outline-offset: 3px; }` — verify the pink outline provides ≥ 3:1 contrast against the gradient background
    - _Requirements: 7.5, 9.6_

---

- [ ] 16. Property-based tests setup and harness
  - [~] 16.1 Add fast-check UMD CDN `<script>` tag (pinned version) and create a `tests.html` file that imports `index.html` logic as a module (or copies extractable pure functions) to run the PBT suite
    - Configure `fc.configureGlobal({ numRuns: 100 })`
    - Annotate every test with `// Feature: cat-petting-clicker, Property N: <description>` per design doc tag format
    - _Requirements: (testing infrastructure)_

---

- [ ] 17. Final polish and validation
  - [~] 17.1 Audit `index.html` for completeness:
    - Verify all 8 reaction CSS classes and keyframes are present and correctly scoped
    - Verify counter reset button has `role="button"` and visible focus ring
    - Verify `#cat-container` has `aspect-square`, shadow/glow ≥ 8px blur
    - Verify `font-family` fallback chain includes system sans-serif
    - _Requirements: 7.2, 7.3, 7.4, 7.5_

  - [~] 17.2 Run through the manual checklist in the design doc:
    - Idle animation on load, particle burst at cursor, counter updates, happiness bar transitions, milestones at 10/50/100/250/500, reduced-motion suppression, keyboard navigation, focus indicators, localStorage persistence across reload
    - _Requirements: 3.1, 8.1, 4.1, 5.3, 6.2, 9.3, 9.2, 7.5, 4.3_

- [~] 18. Final checkpoint — Ensure all tests pass
  - Run the `tests.html` PBT suite and confirm all 10 properties pass with 100 runs each. Verify manual checklist items. Ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; the app is fully functional without them.
- Every task references specific requirements from `requirements.md` for traceability.
- All property-based tests use [fast-check](https://github.com/dubzzz/fast-check) (CDN UMD build); no Node or bundler required.
- Pure logic functions (`pickReaction`, `updateHappinessMeter`, `Storage.load/save`, `checkMilestones`, `spawnParticles`) should be structured so they can be imported or copy-pasted into `tests.html` without a DOM dependency.
- Checkpoints (tasks 10 and 18) ensure incremental validation at natural integration boundaries.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["3.1", "4.1"] },
    { "id": 3, "tasks": ["5.1", "5.2", "6.1", "7.1"] },
    { "id": 4, "tasks": ["5.3", "5.4", "5.5", "6.2", "8.1", "8.2"] },
    { "id": 5, "tasks": ["6.3", "8.3", "8.4", "9.1", "9.2", "9.3"] },
    { "id": 6, "tasks": ["9.4", "11.1", "12.1"] },
    { "id": 7, "tasks": ["12.2", "13.1"] },
    { "id": 8, "tasks": ["13.2", "13.3", "13.4"] },
    { "id": 9, "tasks": ["14.1"] },
    { "id": 10, "tasks": ["14.2", "14.3"] },
    { "id": 11, "tasks": ["15.1", "15.2"] },
    { "id": 12, "tasks": ["16.1"] },
    { "id": 13, "tasks": ["17.1", "17.2"] }
  ]
}
```
