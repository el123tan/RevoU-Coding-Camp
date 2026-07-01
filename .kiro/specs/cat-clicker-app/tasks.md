# Implementation Plan: Cat Clicker App

## Overview

Build a single `index.html` file containing the complete Cat Clicker App — an illustrated cozy-room scene with an interactive chibi cat, click counter, randomized reactions, milestone overlays, localStorage persistence, keyboard accessibility, and CSS-only animations. The implementation proceeds layer by layer: HTML scaffold → CSS styles → pure JS logic → event wiring → accessibility → testing.

## Tasks

- [x] 1. Scaffold the single `index.html` file with base structure
  - Create `index.html` with `<!DOCTYPE html>`, `<html>`, `<head>`, and `<body>` elements
  - Add `<title>Cat Clicker — Pet the Cat!</title>` inside `<head>`
  - Load Tailwind CSS via `<script src="https://cdn.tailwindcss.com"></script>` in `<head>`
  - Add an empty `<style>` block (for custom CSS) and an empty `<script>` block at the bottom of `<body>`
  - Add `height: 100%; overflow: hidden` to `html` and `body`
  - _Requirements: 8.1, 7.5_

- [ ] 2. Build the scene DOM markup
  - [x] 2.1 Create the `#scene` container and `#background` wall element
    - Add `<div id="scene">` with Tailwind classes `relative overflow-hidden mx-auto w-full md:w-[900px]` and inline `aspect-ratio: 4/3`
    - Add `<div id="background">` as absolute fill with `background-color: #f5f0e8`
    - _Requirements: 1.1, 1.2, 6.1_

  - [-] 2.2 Add the window, sky, clouds, and flower curtains inside `#background`
    - Create `<div id="window">` as a bordered div with inner sky `<div>` (`background: #87CEEB`)
    - Add exactly 2 `<div class="cloud">` elements inside the sky pane, styled with `border-radius: 50%` cluster shapes
    - Add two curtain `<div>` elements flanking the window, styled with `radial-gradient` flower pattern over a yellow base
    - _Requirements: 1.3, 1.4_

  - [-] 2.3 Add the floor element
    - Create `<div id="floor">` absolutely positioned at the bottom, `height: 22%`, styled with `repeating-linear-gradient` for oak-plank appearance
    - _Requirements: 1.5_

  - [-] 2.4 Add the cat bed and cat (Petting_Zone) with inline SVG
    - Create `<div id="cat-bed">` centered horizontally with CSS oval/rounded shape and pink blanket inner `<div>`
    - Create `<div id="cat" role="button" tabindex="0" aria-label="Pet the cat" class="relative cursor-pointer ...">` inside `#cat-bed`
    - Insert the full inline SVG cat illustration (body, stripes, head, ears, inner ears, eyes group with `.eye-default` and `.eye-happy hidden`, nose, whiskers, tail, paws) inside `#cat`
    - _Requirements: 1.6, 1.7, 1.8, 7.1, 7.3_

  - [~] 2.5 Add the counter display, floating container, milestone overlay, and aria-live region
    - Add `<div id="counter-display" class="fixed top-4 right-4 z-50 ...">🐾 <span id="counter-value">0</span> Pets</div>`
    - Add `<div id="floating-container" class="absolute inset-0 pointer-events-none overflow-hidden"></div>` inside `#scene`
    - Add `<div id="milestone-overlay" class="hidden fixed inset-0 flex items-center justify-center z-50 ...">` with inner card `<div>`
    - Add `<div id="aria-live" role="status" aria-live="polite" class="sr-only"></div>`
    - _Requirements: 4.1, 4.3, 4.4, 7.6_

- [ ] 3. Write all custom CSS in the `<style>` block
  - [~] 3.1 Define CSS variables, layout, and scene scaling rules
    - Declare CSS custom properties for the color palette
    - Write `#scene` CSS with `width: 900px; aspect-ratio: 4/3; max-width: 100vw; overflow: hidden`
    - Add `@media (max-width: 767px) { #scene { width: 100vw; } }` for mobile scaling
    - _Requirements: 6.1, 6.2, 6.3_

  - [~] 3.2 Define all `@keyframes` animations and animation classes
    - Write `@keyframes cat-bounce` and `.cat--bounce { animation: cat-bounce 400ms ease-out forwards; }`
    - Write `@keyframes float-up` and `.floating-element { animation: float-up 1000ms ease-out forwards; }`
    - Write `@keyframes fade-out` and `.milestone-fade { animation: fade-out 500ms ease-out forwards; }`
    - _Requirements: 2.5, 3.5, 4.4, 8.3_

  - [~] 3.3 Write hover, focus, and happy-eyes CSS rules
    - Write `#cat { filter: brightness(1); transition: filter 150ms ease, box-shadow 150ms ease; }` and `:hover/:focus-visible` glow rule
    - Write `#cat:focus-visible { outline: 2px solid <accessible-color>; }` for keyboard focus indicator
    - Write `#cat.cat--happy-eyes .eye-default { display: none; }` and `#cat.cat--happy-eyes .eye-happy { display: block; }`
    - Write `.eye-happy { display: none; }` default hidden state
    - _Requirements: 5.1, 5.2, 5.3, 7.4_

- [ ] 4. Implement pure JavaScript logic (no DOM side effects)
  - [~] 4.1 Define `AppState`, `REACTION_POOL`, `MILESTONES`, and `MILESTONE_MESSAGES` constants
    - Write `const AppState = { clickCount, lastReactionIndex, animationPlaying, pendingReaction }` singleton
    - Write the full 8-entry `REACTION_POOL` array with all required reaction objects (heart, music-note, star, purr, mrrrow, zzz, fish, happy-eyes)
    - Write `MILESTONES` array and `MILESTONE_MESSAGES` object
    - _Requirements: 3.1, 3.4, 4.4_

  - [ ]* 4.2 Write property test for reaction pool structural validity (Property 4)
    - **Property 4: Reaction Pool Structural Validity**
    - **Validates: Requirements 3.1, 3.2**
    - Use `fast-check` to iterate over `REACTION_POOL` and assert each entry has non-empty `id`, `label`, valid `type`, and at least one of `content` or `catClass`

  - [ ]* 4.3 Write property test for float duration bounds (Property 5)
    - **Property 5: Float Duration Bounds**
    - **Validates: Requirements 3.5**
    - Iterate over `REACTION_POOL` entries where `type === 'float'` and assert `800 ≤ durationMs ≤ 1200`

  - [~] 4.4 Implement `loadCount()` and `saveCount(n)` localStorage adapter functions
    - Write `loadCount()` with `try/catch` around `localStorage.getItem` and `parseInt(..., 10)`, returning 0 on error
    - Write `saveCount(n)` with `try/catch` around `localStorage.setItem`, silently failing on error
    - _Requirements: 4.5, 4.6, 4.7_

  - [ ]* 4.5 Write property test for localStorage round-trip (Property 7)
    - **Property 7: localStorage Round-Trip**
    - **Validates: Requirements 4.5, 4.6**
    - Use `fc.nat()` to generate random non-negative counts; assert `saveCount(n)` then `loadCount()` returns `n`

  - [~] 4.6 Implement `selectReaction(pool, lastIndex)` function
    - Re-sample a random index until it differs from `lastIndex` (handles no-repeat rule)
    - Return the reaction object at the selected index
    - _Requirements: 2.3, 3.3_

  - [ ]* 4.7 Write property test for reaction selection — pool membership and no-repeat (Property 2)
    - **Property 2: Reaction Selection — Pool Membership and No-Repeat**
    - **Validates: Requirements 2.3, 3.3**
    - Use `fc.integer({ min: 2, max: 50 })` for sequence length; assert every selected reaction id exists in `REACTION_POOL` and no two consecutive reactions share the same id

  - [~] 4.8 Implement `checkMilestone(prev, next)` function
    - Return the milestone value M if `prev < M && next >= M` for any M in `MILESTONES`, else return `null`
    - _Requirements: 4.4_

  - [~] 4.9 Implement `handleInteraction()` — the core click/key handler (pure state logic only)
    - Increment `AppState.clickCount`
    - Call `saveCount` and `checkMilestone`
    - Call `selectReaction` and update `AppState.lastReactionIndex`
    - If `animationPlaying`, set `pendingReaction` to the new reaction; else trigger the reaction immediately
    - _Requirements: 2.2, 2.3, 2.6_

  - [ ]* 4.10 Write property test for counter increment invariant (Property 1)
    - **Property 1: Counter Increment Invariant**
    - **Validates: Requirements 2.2, 7.2**
    - Use `fc.nat()` for starting count; set `AppState.clickCount = n`, call `handleInteraction()`, assert `AppState.clickCount === n + 1`

  - [ ]* 4.11 Write property test for reaction queue invariant (Property 3)
    - **Property 3: Reaction Queue Invariant**
    - **Validates: Requirements 2.6**
    - Use `fc.array(fc.nat({ max: 7 }), { minLength: 3 })` to simulate rapid clicks; assert `pendingReaction` always equals the most recent queued reaction and is never non-null when no animation is playing

- [~] 5. Checkpoint — Ensure all logic tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Wire DOM interactions and animation lifecycle
  - [~] 6.1 Initialize the app on `DOMContentLoaded`
    - Read `loadCount()` and set `AppState.clickCount`; update `#counter-value` textContent
    - Attach `pointerdown` listener on `#cat` → calls `handleInteraction()`
    - Attach `keydown` listener on `#cat` for Space/Enter keys → calls `handleInteraction()`
    - _Requirements: 2.1, 4.6, 7.2_

  - [ ]* 6.2 Write property test for keyboard–mouse interaction equivalence (Property 10)
    - **Property 10: Keyboard–Mouse Interaction Equivalence**
    - **Validates: Requirements 7.2**
    - Use `fc.constantFrom('Space', 'Enter')` to simulate keydown; assert the same state transition as a pointer click (count increments, reaction triggers, `animationPlaying` becomes true)

  - [~] 6.3 Implement the reaction renderer — spawn floating elements and speech bubbles
    - For `type === 'float'`: create a `<span>` with `textContent = reaction.content`, add `.floating-element` class, append to `#floating-container`, attach `animationend` listener to remove from DOM
    - For `type === 'bubble'`: create/show a speech bubble `<div>` positioned above `#cat`, transition opacity in, set `setTimeout` to hide after `durationMs`
    - For `type === 'cat-anim'`: add `reaction.catClass` to `#cat`, remove after `durationMs`
    - Add `.cat--bounce` to `#cat` on every reaction; set `AppState.animationPlaying = true`
    - Update `#aria-live` textContent to `reaction.label`
    - _Requirements: 2.4, 2.5, 3.2, 3.5, 7.6_

  - [ ]* 6.4 Write property test for aria-live reaction announcement (Property 11)
    - **Property 11: Aria-Live Reaction Announcement**
    - **Validates: Requirements 7.6**
    - Use `fc.constantFrom(...REACTION_POOL)` to pick a reaction; trigger the renderer, assert `#aria-live` textContent includes `reaction.label` synchronously

  - [~] 6.5 Implement `animationend` handler on `#cat` to drain the reaction queue
    - Remove `.cat--bounce` from `#cat`
    - If `AppState.pendingReaction` exists: play it (call reaction renderer), clear `pendingReaction`
    - Else set `AppState.animationPlaying = false`
    - Add safety timeout (`reaction.durationMs + 200ms`) to drain queue if `animationend` never fires
    - _Requirements: 2.6_

  - [~] 6.6 Implement the counter display update and milestone overlay logic
    - On every `handleInteraction()` call, update `#counter-value` textContent within 50ms
    - When `checkMilestone` returns a value M: inject `MILESTONE_MESSAGES[M]` into `#milestone-overlay`, remove `hidden`, add `milestone-fade` after 2000ms, re-add `hidden` after 500ms fade
    - If overlay is already visible when a new milestone fires, replace content immediately
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 6.7 Write property test for milestone overlay trigger (Property 6)
    - **Property 6: Milestone Overlay Trigger**
    - **Validates: Requirements 4.4**
    - Use `fc.constantFrom(...MILESTONES)` for M; set `AppState.clickCount = M - 1`, call `handleInteraction()`, assert `#milestone-overlay` is visible and contains a non-empty message string

- [~] 7. Checkpoint — Ensure all wiring and integration tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Validate accessibility and visual correctness
  - [~] 8.1 Verify all accessibility attributes and keyboard focus behavior
    - Confirm `#cat` has `tabindex="0"`, `role="button"`, `aria-label="Pet the cat"`
    - Confirm `#aria-live` has `role="status"` and `aria-live="polite"`
    - Confirm `document.title` contains "Cat Clicker"
    - Confirm focus ring is visible with ≥2px outline meeting WCAG 2.1 AA contrast
    - _Requirements: 7.1, 7.3, 7.4, 7.5, 7.6_

  - [~] 8.2 Write unit tests for static DOM invariants and counter display
    - Assert pool contains exactly 8 distinct reaction ids
    - Assert `#counter-display` textContent contains "Pets" and "🐾"
    - Assert exactly 2 `.cloud` elements exist in DOM
    - Assert after a simulated click, `#counter-value` textContent updates and `.cat--bounce` class is present on `#cat`
    - _Requirements: 3.4, 4.2, 4.3, 1.4, 2.5_

- [~] 9. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- All property tests use `fast-check` (`npm install fast-check` in a companion test file); pure logic functions are extracted from the `<script>` block for isolated testing
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at each major phase
- Properties 8 and 9 (viewport layout / responsive scaling) require a browser environment — verify these manually at widths 375, 768, 1024, 1280, 1920 or via Playwright snapshot tests
- The entire deliverable is a single `index.html` file; no build tools, no bundler

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["2.2", "2.3", "2.4"] },
    { "id": 2, "tasks": ["2.5", "3.1"] },
    { "id": 3, "tasks": ["3.2", "3.3", "4.1"] },
    { "id": 4, "tasks": ["4.2", "4.3", "4.4", "4.6", "4.8"] },
    { "id": 5, "tasks": ["4.5", "4.7", "4.9"] },
    { "id": 6, "tasks": ["4.10", "4.11", "6.1"] },
    { "id": 7, "tasks": ["6.2", "6.3"] },
    { "id": 8, "tasks": ["6.4", "6.5", "6.6"] },
    { "id": 9, "tasks": ["6.7", "8.1"] },
    { "id": 10, "tasks": ["8.2"] }
  ]
}
```
