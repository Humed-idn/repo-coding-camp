# Requirements Document

## Introduction

The Cat Petting Clicker is a single-page, high-engagement web application where users interact with a cute, minimalist animated cat by clicking or tapping on it. Each interaction triggers a randomised, delightful reaction — including visual animations, floating emoji particles, sound cues, and a persistently growing affection counter. The experience is built on pure HTML5, CSS (Tailwind + custom), and Vanilla JavaScript, wrapped in a soft glassmorphism aesthetic.

## Glossary

- **App**: The Cat Petting Clicker single-page web application.
- **Cat**: The central animated SVG/pixel-art cat character rendered on the page.
- **Pet Action**: A user-initiated click or tap on the Cat element.
- **Reaction**: A randomised visual and/or audio response triggered by a Pet Action.
- **Affection Counter**: The persistent numeric display tracking the total number of Pet Actions performed by the user.
- **Particle**: A short-lived floating emoji or shape spawned at the click/tap coordinates during a Reaction.
- **Idle Animation**: A looping subtle animation (e.g. slow breathing or tail sway) that plays on the Cat when no Pet Action is occurring.
- **Happiness Meter**: A visual progress bar or indicator reflecting the cumulative Affection Counter value scaled to a defined range.
- **Reaction Pool**: The predefined set of Reactions from which one is selected at random per Pet Action.
- **Milestone**: A specific Affection Counter threshold that triggers a special one-time celebratory Reaction.

---

## Requirements

### Requirement 1: Core Petting Interaction

**User Story:** As a user, I want to click or tap the cat and see an immediate reaction, so that the interaction feels alive and rewarding.

#### Acceptance Criteria

1. WHEN the user clicks or taps the Cat, THE App SHALL trigger exactly one Reaction selected at uniform random from the Reaction Pool, with each Reaction having equal probability of selection before the shuffle algorithm applies.
2. WHEN a Pet Action occurs, THE App SHALL increment the Affection Counter by 1, up to a maximum value of 999,999.
3. WHEN a Pet Action occurs, THE App SHALL begin the triggered Reaction animation within 100 milliseconds of the Pet Action, and the animation SHALL complete within 800 milliseconds of its start.
4. WHEN multiple Pet Actions occur in rapid succession, THE App SHALL queue each Reaction in a first-in-first-out queue with a maximum size of 100, so that no Pet Action is silently dropped.
5. WHILE a Reaction animation is playing, THE App SHALL not begin any new Reaction animation until the current one completes, ensuring at most one Reaction animation is active at any given time.

---

### Requirement 2: Reaction Pool

**User Story:** As a user, I want to see a variety of delightful cat responses when I pet the cat, so that the experience remains engaging over many interactions.

#### Acceptance Criteria

1. THE Reaction Pool SHALL contain a minimum of 8 distinct Reaction types, where each Reaction type has a unique name, animation, and associated Particle emoji set.
2. WHEN a Reaction is selected, THE App SHALL display the Reaction-specific animation on the Cat within 100 milliseconds of the Pet Action, running the animation for a duration between 800 milliseconds and 3000 milliseconds before the animation ends.
3. WHEN a Reaction is selected, THE App SHALL spawn between 3 and 7 Particles at the Pet Action coordinates, each Particle being a thematically appropriate emoji (e.g. ♥, ✨, 💫, 🐾, 😸, 💕, 🌸, ⭐), with each Particle visible for between 500 milliseconds and 1500 milliseconds before disappearing.
4. WHEN a Reaction animation completes, THE App SHALL transition the Cat to the Idle Animation state within 100 milliseconds.
5. THE App SHALL use a weighted-random or shuffle algorithm such that no single Reaction appears more than twice within any consecutive sequence of 5 Pet Actions, evaluated as a sliding window updated after each Pet Action.
6. IF a Reaction animation is interrupted by a new Pet Action before it completes, THEN THE App SHALL immediately end the current Reaction animation, transition to the Idle Animation state, and begin the newly selected Reaction animation within 100 milliseconds of the interrupting Pet Action.

---

### Requirement 3: Idle Animation

**User Story:** As a user, I want the cat to appear alive and charming even when I am not interacting with it, so that the page feels warm and inviting.

#### Acceptance Criteria

1. WHILE no Pet Action is being processed, THE Cat SHALL continuously display the Idle Animation loop (gentle up-down float or slow tail sway) with a consistent cycle duration between 2 and 4 seconds, using the same variant for the duration of the session.
2. WHEN a Pet Action begins, THE App SHALL pause the Idle Animation at its current frame and display the Reaction animation for the duration of the active Reaction.
3. WHEN the active Reaction completes, THE App SHALL resume the Idle Animation from its first frame within 100 milliseconds.
4. IF a second Pet Action occurs while a Reaction animation is already active, THEN THE App SHALL queue the new Pet Action and not restart the Idle Animation until all queued Reactions have completed.

---

### Requirement 4: Affection Counter Display

**User Story:** As a user, I want to see how many times I have petted the cat, so that I can feel a sense of growing affection and progression.

#### Acceptance Criteria

1. THE App SHALL display the Affection Counter as a non-negative integer permanently visible on the page.
2. WHEN the Affection Counter value changes, THE App SHALL animate the counter update with a scale transition from 1.0 to 1.3 and back to 1.0, completing within 300 milliseconds.
3. THE App SHALL persist the Affection Counter value in the browser's localStorage under the key `catPettingCount` so that the count survives page reloads.
4. WHEN the App loads, THE App SHALL read `catPettingCount` from localStorage and initialise the Affection Counter to that stored value, or 0 if no stored value exists.
5. IF the value read from localStorage under `catPettingCount` is not a non-negative integer, THEN THE App SHALL initialise the Affection Counter to 0.

---

### Requirement 5: Happiness Meter

**User Story:** As a user, I want a visual indicator of my cat's overall happiness level, so that I feel motivated to keep petting.

#### Acceptance Criteria

1. THE App SHALL display a Happiness Meter as a horizontal progress bar with a fill element whose width represents the current happiness percentage.
2. THE App SHALL calculate the Happiness Meter fill percentage as `floor(log(n + 1) / log(501) × 100)`, where `n` is the current Affection Counter value, mapping 0 Pet Actions to 0% and 500 Pet Actions to 100%.
3. WHEN the Affection Counter changes, THE App SHALL update the Happiness Meter fill width with a CSS transition of 400 milliseconds.
4. WHEN the Happiness Meter reaches 100%, THE App SHALL hold the fill at 100% for all subsequent Pet Actions without visual overflow.
5. WHEN the App is first loaded and the Affection Counter is 0, THE App SHALL display the Happiness Meter at 0% fill.

---

### Requirement 6: Milestone Celebrations

**User Story:** As a user, I want special celebrations at key petting milestones, so that long-term engagement feels meaningful and rewarding.

#### Acceptance Criteria

1. THE App SHALL define Milestones at Affection Counter values of 10, 50, 100, 250, and 500.
2. WHEN the Affection Counter reaches a Milestone value for the first time, THE App SHALL trigger a full-screen Particle burst (covering the entire viewport) of at least 20 Particles, and shall display a celebratory on-screen text message visible for at least 3000 milliseconds.
3. WHEN a Milestone Reaction plays, THE App SHALL complete it within 2000 milliseconds before re-enabling normal petting input.
4. THE App SHALL store triggered Milestone values in localStorage under the key `catMilestonesReached` so that each Milestone celebration fires only once per user.
5. IF the Affection Counter value jumps past a Milestone (e.g. from 9 to 11), THEN THE App SHALL still trigger the skipped Milestone celebration.

---

### Requirement 7: Visual Aesthetic — Soft Glassmorphism

**User Story:** As a user, I want a visually cohesive, kawaii aesthetic throughout the app, so that the experience feels charming and polished.

#### Acceptance Criteria

1. THE App SHALL render a full-viewport background using a CSS linear gradient from `#FFDEE9` to `#B5FFFC` (top-left to bottom-right).
2. THE App SHALL apply a playful, rounded sans-serif font (Quicksand or Varela Round, loaded via Google Fonts) to all visible text, and IF the Google Fonts resource fails to load, THEN THE App SHALL fall back to a system sans-serif font so that all text remains legible.
3. THE App SHALL style all UI containers (counter bubble, Happiness Meter wrapper, label text cards) using Tailwind utility classes `bg-white/30`, `backdrop-blur-lg`, `rounded-full`, and `border border-white/40`.
4. THE Cat element SHALL be centered on the page using Tailwind's `aspect-square` utility and SHALL render a shadow or glow effect beneath it with a blur radius of at least 8px and an opacity between 20% and 60%, visually separating the Cat from the background.
5. WHEN any interactive element (Cat, counter reset button) receives keyboard focus, THE App SHALL display a focus indicator with a minimum outline width of 2px and a contrast ratio of at least 3:1 between the focus indicator color and the adjacent background color.

---

### Requirement 8: Particle Animation System

**User Story:** As a user, I want the emoji particles that appear when I pet the cat to feel bubbly and satisfying, so that each click has tactile visual feedback.

#### Acceptance Criteria

1. WHEN Particles are spawned, THE App SHALL position each Particle at the Pet Action coordinates (click/tap x, y) relative to the viewport.
2. WHEN Particles are spawned, THE App SHALL animate each Particle along a unique randomised arc — drifting upward between 40px and 120px and laterally between −50px and +50px — over a duration between 600 and 1000 milliseconds.
3. WHEN a Particle animation completes, THE App SHALL remove the Particle element from the DOM to prevent accumulation.
4. THE App SHALL cap the total number of simultaneously active Particles at 30; IF a new Pet Action would cause the active Particle count to exceed 30, THEN THE App SHALL remove the oldest active Particles until the count is at or below 30 before spawning new Particles.
5. WHEN Particles animate, THE App SHALL fade each Particle from opacity 1.0 to 0.0 linearly over the final 30% of its animation duration.

---

### Requirement 9: Accessibility and Performance

**User Story:** As a user with assistive technology or a low-powered device, I want the app to be usable and performant, so that I am not excluded from the experience.

#### Acceptance Criteria

1. THE Cat element SHALL have an ARIA label of "Pet the cat" and a `role="button"` attribute so that screen readers can announce and activate the Pet Action.
2. WHEN the user activates the Cat via the keyboard (Enter or Space key), THE App SHALL trigger a Pet Action that increments the Affection Counter by 1 and triggers the same visual Reaction feedback as a click interaction.
3. WHERE the user has enabled the `prefers-reduced-motion` media query, THE App SHALL disable Particle animations and Cat bounce animations, substituting a simple opacity fade of no more than 150 milliseconds for all motion effects.
4. THE App SHALL achieve a Lighthouse Performance score of 90 or above when tested using the Lighthouse CLI tool in simulated desktop mode with no browser extensions active.
5. THE App SHALL reach Time to Interactive within 3 seconds of navigation start when tested on a connection with a 10 Mbps download speed and a round-trip latency of 40 milliseconds or less.
6. WHEN any interactive element receives keyboard focus, THE App SHALL display a focus indicator visible to sighted users with a contrast ratio of at least 3:1 against the adjacent background, per WCAG 2.1 Success Criterion 1.4.11.
