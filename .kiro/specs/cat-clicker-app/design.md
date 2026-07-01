# Design Document — Cat Clicker App

## Overview

The Cat Clicker App is a single-page, zero-dependency web application delivered as a single HTML file. It presents an illustrated cozy room scene containing an animated, interactive chibi-style cat. The user pets the cat by clicking or pressing a key; each interaction triggers a randomized visual reaction and increments a persistent counter.

**Tech stack constraints:**
- HTML5 (single file, no build tools)
- Tailwind CSS (loaded via CDN `<script>` tag)
- Vanilla JavaScript (ES2020, no frameworks)
- No external image assets — all visuals are CSS shapes, inline SVG, or HTML/CSS composition
- No JavaScript-driven animation — CSS `transition` and `@keyframes` only
- No server; works as a local `file://` or static-host page

**Core user loop:**

```
User clicks cat → counter increments → reaction selected (no repeat) → CSS animation plays
       ↑                                                                          ↓
  localStorage persist ←────────────────────────────────────────── floating element fades out
```

---

## Architecture

The entire application lives in one `index.html` file, structured into four logical layers:

```
┌─────────────────────────────────────────────────────┐
│  Presentation Layer  (HTML markup + Tailwind classes)│
│  • Scene DOM tree                                    │
│  • SVG inline illustrations                          │
│  • Tailwind utility classes                          │
├─────────────────────────────────────────────────────┤
│  Style Layer  (<style> block — custom CSS only)      │
│  • @keyframes definitions                            │
│  • CSS variables for palette                         │
│  • Responsive scaling via CSS container queries /    │
│    viewport-relative units                           │
├─────────────────────────────────────────────────────┤
│  State Layer  (module-pattern JS in <script>)        │
│  • AppState singleton                                │
│  • localStorage adapter                              │
│  • Reaction queue (current + at-most-1 pending)      │
├─────────────────────────────────────────────────────┤
│  Interaction Layer  (event listeners)                │
│  • pointerdown on Petting_Zone                       │
│  • keydown (Space / Enter) on Petting_Zone           │
│  • animationend listeners for queue drain            │
└─────────────────────────────────────────────────────┘
```

There are no module imports, no bundler, no CDN dependencies beyond Tailwind's CDN `<script>` tag.

### File layout (single `index.html`)

```
<head>
  <title>Cat Clicker — Pet the Cat!</title>
  <script src="https://cdn.tailwindcss.com"></script>   <!-- Tailwind CDN -->
  <style>  /* custom keyframes + CSS vars */  </style>
</head>
<body>
  <!-- Scene markup -->
  <!-- Aria-live region -->
  <script>  /* all JS */  </script>
</body>
```

---

## Components and Interfaces

### 2.1 Scene

The scene is a fixed-aspect-ratio container that holds all visual sub-components. It uses a CSS `aspect-ratio: 4/3` so it fills the viewport on desktop and scales down on mobile.

```
#scene  (relative-positioned outer container)
├── #background     (beige wall, absolute fill)
│   └── #window     (flower-curtained window + sky + 2 clouds)
├── #floor          (oak-plank strip, absolute bottom 20–25%)
├── #cat-bed        (bed + pink blanket, centered horizontally)
│   └── #cat        (the interactive cat, Petting_Zone)
│       ├── .cat-body   (CSS+SVG orange striped torso)
│       ├── .cat-head   (occupies ≥40% of total cat height)
│       ├── .cat-eyes   (can switch to happy-arc variant via class)
│       └── .cat-tail
├── #floating-container  (floats spawn here)
├── #counter-display    (fixed, top-left or top-right; never overlaps cat)
├── #milestone-overlay  (hidden by default; shows celebratory message)
└── #aria-live          (sr-only, role="status" aria-live="polite")
```

### 2.2 Cat (Petting_Zone)

`#cat` doubles as the Petting_Zone. Key attributes:

```html
<div id="cat"
     role="button"
     tabindex="0"
     aria-label="Pet the cat"
     class="relative cursor-pointer ...">
  <!-- cat SVG/CSS composition -->
</div>
```

States driven by CSS classes toggled by JS:

| CSS class applied to `#cat` | Meaning |
|---|---|
| `cat--bounce` | Short bounce animation (200–600 ms) |
| `cat--happy-eyes` | Eyes curve into arcs |
| `cat--hover` | Brightness/glow (also handled by `:hover` + `:focus-visible`) |

### 2.3 Window / Sky Component

Rendered purely with CSS + inline SVG:

- **Frame**: a bordered `<div>` with rounded corners
- **Sky pane**: solid `#87CEEB` background fill
- **Clouds**: two `<div>` elements styled with `border-radius` pill shapes (no images)
- **Curtains**: two `<div>` flanking the window, colored with a repeating CSS pattern using `radial-gradient` for flower motif

### 2.4 Floor Component

A horizontal strip at the bottom of the scene using a CSS `repeating-linear-gradient` to simulate oak wood planks.

### 2.5 Cat Bed Component

The bed is a CSS oval/rounded shape. The pink blanket is a `<div>` with `border-radius` and a subtle CSS texture gradient. The cat sits inside this boundary.

### 2.6 Floating Element Component

Dynamically created DOM nodes injected by JS when a reaction fires. Each float:

- Is a single `<span>` or inline SVG inserted into `#floating-container`
- Has a CSS class that triggers an `@keyframes` animation (`float-up`) driving `transform: translateY` and `opacity` from 1 → 0
- Is removed from the DOM via an `animationend` listener after the animation completes

### 2.7 Counter Display

```html
<div id="counter-display" class="fixed top-4 right-4 ...">
  🐾 <span id="counter-value">0</span> Pets
</div>
```

Always on screen; `z-index` above scene elements; never overlaps cat/bed.

### 2.8 Milestone Overlay

```html
<div id="milestone-overlay" class="hidden fixed inset-0 flex items-center justify-center ...">
  <div class="bg-white rounded-xl shadow-lg px-8 py-4 text-2xl">
    <!-- milestone text injected by JS -->
  </div>
</div>
```

Lifecycle: JS injects text → removes `hidden` → adds `fade-out` class after 2000 ms → overlay auto-hides after 500 ms transition.

### 2.9 Aria-Live Region

```html
<div id="aria-live" role="status" aria-live="polite" class="sr-only"></div>
```

JS sets `textContent` on each reaction trigger so screen readers announce the reaction type.

---

## Data Models

### AppState (singleton object in JS)

```js
const AppState = {
  clickCount: 0,          // Number — restored from localStorage on init
  lastReactionIndex: -1,  // Number — index of last played reaction; -1 = none
  animationPlaying: false,// Boolean — true while cat bounce CSS animation runs
  pendingReaction: null,  // Reaction | null — at most 1 queued reaction
};
```

### Reaction (plain object)

```js
/**
 * @typedef {Object} Reaction
 * @property {string}  id          — unique identifier, e.g. "heart"
 * @property {string}  label       — human-readable name for aria-live, e.g. "Heart"
 * @property {'float'|'bubble'|'cat-anim'} type
 * @property {string}  [content]   — HTML/SVG string for float or bubble text
 * @property {string}  [catClass]  — CSS class to add to #cat during this reaction
 * @property {number}  durationMs  — how long the reaction animation lasts (ms)
 */
```

### ReactionPool (array)

```js
const REACTION_POOL = [
  { id: 'heart',      label: 'Heart',       type: 'float',    content: '❤️',   durationMs: 1000 },
  { id: 'music-note', label: 'Purring',     type: 'float',    content: '♪',    durationMs: 1000 },
  { id: 'star',       label: 'Star',        type: 'float',    content: '⭐',   durationMs: 1000 },
  { id: 'purr',       label: 'Purrrr~',     type: 'bubble',   content: 'Purrrr~', durationMs: 1500 },
  { id: 'mrrrow',     label: 'Mrrrow!',     type: 'bubble',   content: 'Mrrrow!', durationMs: 1500 },
  { id: 'zzz',        label: 'Sleepy Zzz',  type: 'float',    content: 'Zzz',  durationMs: 1000 },
  { id: 'fish',       label: 'Fish',        type: 'float',    content: '🐟',   durationMs: 1000 },
  { id: 'happy-eyes', label: 'Happy Eyes',  type: 'cat-anim', catClass: 'cat--happy-eyes', durationMs: 800 },
];
```

### localStorage Schema

```
Key:   "cat-clicker-count"
Value: JSON-serialized integer string, e.g. "42"
```

Write: `localStorage.setItem('cat-clicker-count', String(AppState.clickCount))` on every increment.  
Read: `parseInt(localStorage.getItem('cat-clicker-count') ?? '0', 10)` on load; wrapped in `try/catch`.

### Milestone Table

```js
const MILESTONES = [10, 50, 100, 500, 1000];
const MILESTONE_MESSAGES = {
  10:   '🎉 10 Pets! The cat approves!',
  50:   '😻 50 Pets! You\'re a cat whisperer!',
  100:  '💯 100 Pets! Legendary petter!',
  500:  '🏆 500 Pets! The cat is purring non-stop!',
  1000: '👑 1000 Pets! You are the Cat Clicker Champion!',
};
```

---


## CSS / SVG Illustration Approach

All visuals are produced without external image files. The strategy divides the scene into three rendering techniques:

### Technique A — CSS-only shapes (backgrounds, floor, clouds, bed)

Used for simple geometric regions where gradients and border-radius are sufficient.

| Element | Technique |
|---|---|
| Beige wall | `background-color: #f5f0e8` on `#background` |
| Oak floor | `repeating-linear-gradient` with 3-4 color stops simulating wood grain, applied to `#floor` |
| Clouds | `<div>` with `border-radius: 50%` cluster, white background, no border |
| Cat bed body | `<div>` with `border-radius: 60% 60% 40% 40%` oval, warm brown tones |
| Pink blanket | `<div>` with `border-radius` + subtle `linear-gradient` in pink tones |
| Window frame | `<div>` with thick `border`, inner sky `<div>` in `#87CEEB` |

### Technique B — CSS pattern + gradient (curtains, floor planks)

| Element | Technique |
|---|---|
| Flower curtains | `radial-gradient` (small circles) layered over a yellow base using `background-image` with `background-size` repeat to simulate a flower tile pattern |
| Wood planks | `repeating-linear-gradient` at a shallow angle to simulate plank lines; `#floor` uses two overlapping gradients for a realistic wood look |

### Technique C — Inline SVG (cat character)

The cat is the most detailed element and is best rendered as an inline SVG placed inside `#cat`. This gives precise shape control for the striped chibi body.

```
<svg id="cat-svg" viewBox="0 0 120 160" ...>
  <!-- Body: rounded ellipse, orange (#E8851A) base -->
  <ellipse cx="60" cy="100" rx="42" ry="38" fill="#E8851A"/>
  <!-- Stripes: darker orange stripes via <path> or <rect> with clip-path -->
  <path d="..." fill="#C06A10" clip-path="url(#body-clip)"/>
  <!-- Head: large circle, ≥40% of total SVG height = ≥64px in 160px viewBox -->
  <circle cx="60" cy="52" r="38" fill="#E8851A"/>
  <!-- Ears: two triangular <polygon> elements -->
  <polygon points="28,24 18,2 48,20" fill="#E8851A"/>
  <polygon points="92,24 102,2 72,20" fill="#E8851A"/>
  <!-- Inner ears: smaller pink triangles -->
  <polygon points="31,22 22,6 44,19" fill="#FFB6C1"/>
  <polygon points="89,22 98,6 76,19" fill="#FFB6C1"/>
  <!-- Eyes: default = two filled circles; happy = arc paths via CSS class toggle -->
  <g id="cat-eyes">
    <circle class="eye-default" cx="44" cy="50" r="6" fill="#2C1810"/>
    <circle class="eye-default" cx="76" cy="50" r="6" fill="#2C1810"/>
    <path class="eye-happy hidden" d="M38,50 Q44,42 50,50" stroke="#2C1810" stroke-width="3" fill="none"/>
    <path class="eye-happy hidden" d="M70,50 Q76,42 82,50" stroke="#2C1810" stroke-width="3" fill="none"/>
  </g>
  <!-- Nose: small triangle -->
  <polygon points="57,63 63,63 60,67" fill="#FFB6C1"/>
  <!-- Whiskers: <line> elements -->
  <line x1="15" y1="65" x2="48" y2="67" stroke="#8B6914" stroke-width="1.5"/>
  <line x1="72" y1="67" x2="105" y2="65" stroke="#8B6914" stroke-width="1.5"/>
  <!-- Tail: bezier <path> curling to the side -->
  <path d="M95,130 Q130,110 125,85 Q120,70 108,78" stroke="#E8851A" stroke-width="10" fill="none" stroke-linecap="round"/>
  <!-- Paws: small ellipses at bottom of body -->
  <ellipse cx="42" cy="135" rx="14" ry="8" fill="#E8851A"/>
  <ellipse cx="78" cy="135" rx="14" ry="8" fill="#E8851A"/>
</svg>
```

The happy-eyes state is toggled by the `cat--happy-eyes` class on `#cat`, which uses a CSS rule to hide `.eye-default` and show `.eye-happy`:

```css
#cat.cat--happy-eyes .eye-default { display: none; }
#cat.cat--happy-eyes .eye-happy   { display: block; }
```

---

## Animation Strategy

All animations use CSS `@keyframes` or `transition` exclusively. No JavaScript touches animation frame state.

### Keyframes defined in `<style>`

```css
/* Cat bounce — triggered by adding .cat--bounce class */
@keyframes cat-bounce {
  0%   { transform: translateY(0) scale(1); }
  30%  { transform: translateY(-12px) scale(1.05, 0.95); }
  55%  { transform: translateY(0) scale(0.97, 1.03); }
  75%  { transform: translateY(-5px) scale(1.02, 0.98); }
  100% { transform: translateY(0) scale(1); }
}
.cat--bounce { animation: cat-bounce 400ms ease-out forwards; }

/* Float-up — applied to each spawned floating element */
@keyframes float-up {
  0%   { transform: translateY(0);    opacity: 1; }
  100% { transform: translateY(-80px); opacity: 0; }
}
.floating-element { animation: float-up 1000ms ease-out forwards; }

/* Milestone overlay fade-out */
@keyframes fade-out {
  0%   { opacity: 1; }
  100% { opacity: 0; }
}
.milestone-fade { animation: fade-out 500ms ease-out forwards; }

/* Hover glow — CSS transition, no keyframe needed */
#cat {
  filter: brightness(1);
  transition: filter 150ms ease, box-shadow 150ms ease;
}
#cat:hover, #cat:focus-visible {
  filter: brightness(1.15);
  box-shadow: 0 0 12px 4px rgba(255, 200, 80, 0.7);
}
```

### Animation lifecycle (JS orchestration)

JS only toggles CSS classes and listens for `animationend` — it never calls `requestAnimationFrame` or `setTimeout` for frame-level work.

```
click
  └── add .cat--bounce to #cat
  └── spawn floating element (or show speech bubble)
  └── set AppState.animationPlaying = true

animationend on #cat
  └── remove .cat--bounce
  └── if pendingReaction exists → play it, clear pendingReaction
  └── else set AppState.animationPlaying = false

animationend on .floating-element
  └── remove element from DOM
```

### Speech bubble reactions

Speech bubbles are `<div>` elements absolutely positioned above the cat, toggled visible/hidden via CSS class. They use a CSS `transition` on `opacity` (0 → 1 over 150 ms) and are hidden again after `durationMs` via a `setTimeout` that only removes a CSS class (no animation frame work).

### CSS performance considerations

All animated properties are `transform` and `opacity` — both compositable on the GPU, ensuring the 60fps requirement is met without layout thrashing. The `filter` property (brightness/glow) is also GPU-composited in modern browsers.

---

## Responsive Layout Approach

### Strategy: scale-down on mobile, fixed on desktop

The scene is designed at a reference size of approximately 900×675px (4:3 aspect ratio). On desktop (≥768px), it renders at its fixed reference size, centered in the viewport. On mobile (<768px), it scales to fill 100vw while maintaining aspect ratio.

### Implementation

```css
/* Scene container */
#scene {
  position: relative;
  width: 900px;
  aspect-ratio: 4 / 3;
  max-width: 100vw;        /* prevents overflow on any viewport */
  overflow: hidden;
}

/* Scale down on mobile */
@media (max-width: 767px) {
  #scene {
    width: 100vw;
  }
}
```

Inside the scene, all child elements use percentage-based positioning and sizing (e.g., `width: 30%`, `left: 35%`) so they scale proportionally when the scene itself shrinks.

The Click_Counter is `position: fixed` outside the scene, so it always stays visible in the viewport corner regardless of scene scaling.

### Tailwind utility mapping

| Concept | Tailwind classes |
|---|---|
| Scene container | `relative overflow-hidden mx-auto` |
| Scene mobile full-width | `w-full md:w-[900px]` |
| Background fill | `absolute inset-0` |
| Floor strip | `absolute bottom-0 w-full h-[22%]` |
| Counter fixed position | `fixed top-4 right-4 z-50` |
| Milestone overlay | `fixed inset-0 flex items-center justify-center z-50` |
| Cat bed centering | `absolute left-1/2 -translate-x-1/2` |
| SR-only aria-live | `sr-only` |

### No-scroll guarantee

`#scene` has `overflow: hidden`. The `<body>` and `<html>` are set to `height: 100%; overflow: hidden` to prevent any scroll. The counter is `fixed`, not in the document flow.

---

## Correctness Properties


*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

---

### Property 1: Counter Increment Invariant

*For any* starting click count N (where N ≥ 0), after the user triggers a single interaction on the Petting_Zone (click, Space, or Enter), the Click_Counter value SHALL equal N + 1.

**Validates: Requirements 2.2, 7.2**

---

### Property 2: Reaction Selection — Pool Membership and No-Repeat

*For any* sequence of N consecutive interactions (N ≥ 2) on the Petting_Zone, every selected Reaction SHALL have an `id` that exists in `REACTION_POOL`, AND no two consecutively selected Reactions SHALL share the same `id`.

**Validates: Requirements 2.3, 3.3**

---

### Property 3: Reaction Queue Invariant

*For any* sequence of 3 or more rapid clicks while a reaction animation is currently playing, `AppState.pendingReaction` SHALL always equal the most recently triggered Reaction, never an earlier one, and its value SHALL never be non-null while no animation is playing.

**Validates: Requirements 2.6**

---

### Property 4: Reaction Pool Structural Validity

*For every* Reaction object in `REACTION_POOL`, it SHALL have a non-empty `id`, a non-empty `label`, a `type` that is one of `'float'`, `'bubble'`, or `'cat-anim'`, and at least one of `content` (non-empty string) or `catClass` (non-empty string) SHALL be present.

**Validates: Requirements 3.1, 3.2**

---

### Property 5: Float Duration Bounds

*For every* Reaction in `REACTION_POOL` with `type === 'float'`, its `durationMs` value SHALL satisfy `800 ≤ durationMs ≤ 1200`.

**Validates: Requirements 3.5**

---

### Property 6: Milestone Overlay Trigger

*For any* milestone value M in `MILESTONES` (10, 50, 100, 500, 1000), when `AppState.clickCount` transitions from M − 1 to M via an increment, the `#milestone-overlay` element SHALL transition from hidden to visible and SHALL contain a non-empty celebratory message string.

**Validates: Requirements 4.4**

---

### Property 7: localStorage Round-Trip

*For any* non-negative integer N representing the click count: (a) after incrementing the counter to N + 1, `localStorage.getItem('cat-clicker-count')` SHALL equal `String(N + 1)`; and (b) for any integer value V stored in localStorage under key `'cat-clicker-count'`, calling the state initialization function SHALL set `AppState.clickCount` to `parseInt(V, 10)`.

**Validates: Requirements 4.5, 4.6**

---

### Property 8: Viewport Layout — No Overflow and Element Visibility

*For any* viewport width W in the range [375, 1920] (pixels), after the scene renders: (a) `document.body.scrollWidth` SHALL NOT exceed W (no horizontal overflow); and (b) the bounding rectangles of `#cat`, `#counter-display`, and `#cat-bed` SHALL be fully contained within the viewport bounds (no clipping).

**Validates: Requirements 6.1, 6.5**

---

### Property 9: Responsive Scaling Rule

*For any* viewport width W: (a) if W < 768, the rendered width of `#scene` SHALL equal W (100vw); (b) if W ≥ 768, the rendered dimensions of `#scene` and all contained elements SHALL remain constant regardless of further increases in W up to 1920px.

**Validates: Requirements 6.2, 6.3**

---

### Property 10: Keyboard–Mouse Interaction Equivalence

*For any* application state S, triggering the Petting_Zone via a Space or Enter keydown event SHALL produce an identical state transition as a pointer click event: `clickCount` SHALL increment by 1, a Reaction SHALL be selected and triggered, and `AppState.animationPlaying` SHALL become true.

**Validates: Requirements 7.2**

---

### Property 11: Aria-Live Reaction Announcement

*For any* Reaction R selected from `REACTION_POOL` and triggered by user interaction, the `textContent` of `#aria-live` SHALL be updated to include `R.label` within the same synchronous execution context as the reaction trigger.

**Validates: Requirements 7.6**

---

## Error Handling

### localStorage Failure

The `localStorage` read on page load and write on each increment are wrapped in a `try/catch`. If `localStorage` is unavailable (private-browsing restrictions, storage quota exceeded, or security policy), the app silently catches the error, initializes `clickCount` to 0, and continues operating. No error is surfaced to the user.

```js
function loadCount() {
  try {
    const stored = localStorage.getItem('cat-clicker-count');
    return stored !== null ? parseInt(stored, 10) : 0;
  } catch {
    return 0;
  }
}

function saveCount(n) {
  try {
    localStorage.setItem('cat-clicker-count', String(n));
  } catch {
    // silent fail — count still works in-memory
  }
}
```

### Reaction Selection Edge Cases

- **Empty pool**: guarded at initialization; the pool is a compile-time constant with 8 entries, so this cannot occur at runtime.
- **Single-entry pool**: if `lastReactionIndex` is the only index, the no-repeat rule would loop infinitely. The pool must have ≥ 2 entries — enforced by the 8-entry constant.
- **Random index equals last**: re-sample until a different index is chosen (bounded by pool size, O(pool_size) in worst case).

### Animation Queue

If `animationend` never fires (e.g., the animation is cancelled externally), the queue would stall. A safety timeout is set equal to `reaction.durationMs + 200ms` to drain the queue if `animationend` does not fire within the expected window.

### DOM Injection of Floating Elements

All float `content` strings in `REACTION_POOL` are compile-time constants (emoji characters and plain text). They are injected via `textContent` assignment, not `innerHTML`, preventing any XSS vector.

---

## Testing Strategy

### Overview

Testing is split into two complementary layers:

1. **Unit / Example tests** — specific behavior, static invariants, edge cases
2. **Property-based tests** — universal properties across many generated inputs

Since the app is vanilla JS with no module system, tests will use a lightweight in-browser test harness or be extracted into a companion test file that imports the pure logic functions (counter arithmetic, reaction selection, localStorage adapter, queue logic) as CommonJS/ES modules for testing with **fast-check** (JavaScript property-based testing library).

### Property-Based Testing Library

**fast-check** (npm: `fast-check`) will be used for all property-based tests.

Each property test runs a minimum of **100 iterations** (fast-check default is 100, can be configured via `{ numRuns: 100 }`).

### Pure Logic Extraction

The following functions are pure (no DOM side effects) and are extracted for isolated testing:

| Function | Inputs | Output |
|---|---|---|
| `selectReaction(pool, lastIndex)` | array, number | Reaction object |
| `loadCount()` | — (reads localStorage mock) | number |
| `saveCount(n)` | number | void |
| `isValidReaction(r)` | Reaction | boolean |
| `checkMilestone(prev, next)` | number, number | number \| null |

### Unit Tests (Example-Based)

| Test | Criteria |
|---|---|
| Pool contains exactly the 8 required reaction ids | 3.4 |
| `#cat` has `tabindex="0"`, `role="button"`, `aria-label="Pet the cat"` | 7.1, 7.3 |
| `document.title` contains "Cat Clicker" | 7.5 |
| `#counter-display` textContent contains "Pets" and "🐾" | 4.3 |
| After click, `#counter-value` updates synchronously | 4.2 |
| After click, `cat--bounce` class is present on `#cat` | 2.5 |
| Exactly 2 `.cloud` elements exist in DOM | 1.4 |
| `#aria-live` has `aria-live="polite"` | 7.6 |

### Property-Based Tests (fast-check)

Each property test is tagged with a comment referencing its design property.

```
// Feature: cat-clicker-app, Property 1: Counter Increment Invariant
fc.assert(fc.property(fc.nat(), (n) => {
  AppState.clickCount = n;
  handleInteraction();
  return AppState.clickCount === n + 1;
}));
```

| Property Test | Design Property | Arbitraries Used |
|---|---|---|
| Counter increment invariant | Property 1 | `fc.nat()` (starting count) |
| Reaction pool membership + no-repeat | Property 2 | `fc.integer({ min: 2, max: 50 })` (sequence length) |
| Reaction queue invariant | Property 3 | `fc.array(fc.nat({ max: 7 }), { minLength: 3 })` (click indices) |
| Reaction pool structural validity | Property 4 | *(iterate over REACTION_POOL)* |
| Float duration bounds | Property 5 | *(iterate over REACTION_POOL filter float)* |
| Milestone overlay trigger | Property 6 | `fc.constantFrom(...MILESTONES)` |
| localStorage round-trip | Property 7 | `fc.nat()` (stored count value) |
| Keyboard–mouse equivalence | Property 10 | `fc.constantFrom('Space', 'Enter')` (key name) |
| Aria-live announcement | Property 11 | `fc.constantFrom(...REACTION_POOL)` (reaction object) |

### Responsive / Visual Tests

Properties 8 and 9 (viewport layout) require a browser environment with real layout. They are implemented as browser-based snapshot tests using Playwright or manual Lighthouse audits:

- Run at viewport widths: 375, 414, 768, 1024, 1280, 1920
- Assert no `scrollWidth` overflow
- Assert bounding rect of `#cat`, `#counter-display`, `#cat-bed` are within viewport

### Accessibility Testing

- **Automated**: axe-core run on the rendered page (checks ARIA roles, labels, color contrast, focus management)
- **Manual**: keyboard-only navigation walkthrough; screen reader announcement spot-check
- Note: Full WCAG 2.1 AA validation requires manual testing with assistive technologies

