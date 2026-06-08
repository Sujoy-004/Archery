# Pitfalls Research

**Domain:** Mobile HTML5 Canvas 2D arcade game (archery drag-to-shoot)
**Researched:** 2026-06-08
**Confidence:** HIGH — verified against Context7 documentation, MDN, web.dev, real-device benchmarks, and community post-mortems

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

---

### Pitfall 1: Per-Frame Object Allocation Triggering GC Stutter

**What goes wrong:**
The game loop allocates new objects (Vec2 positions, arrow trajectories, particle arrays, collision results) every frame. After 1–3 seconds of gameplay, V8's garbage collector pauses the main thread for 10–50ms, dropping 1–3 frames. On low-end Android phones with Hermes Hades GC, pauses are shorter but still non-zero — and at 18,000+ allocations/second (300 particles × 60fps), the young generation fills every ~2 seconds.

**Why it happens:**
- Natural autocomplete produces `Vec2.add(a, b)` which returns a fresh `{x, y}` object
- Array methods like `.filter()`, `.map()`, `.splice()` create intermediate arrays
- `new` keyword inside update loops (projectiles, particles, trail segments)
- Closures inside hot loops create allocation sites that the GC must trace
- JS developers coming from non-game backgrounds aren't trained to think about allocation budgets

**How to avoid:**
- Use plain `{x, y}` objects with an `out` parameter pattern instead of Vec2 classes that allocate on construction
- Implement a generic `Pool<T>` prewarmed at module/load time — one pool for particles, one for sprite commands, one for trail segments
- Use swap-and-pop release (O(1)) instead of `.splice()` to remove dead objects
- Reset arrays with `.length = 0` instead of reassigning `= []`
- Use `ctx.setTransform()` instead of creating new transform matrices
- Prefer `for` loops over `.filter().map()` chains in update paths
- Use `Math.floor()` on positions to avoid sub-pixel rendering, which also reduces GPU blending work

**Warning signs:**
- Sawtooth memory pattern in Chrome DevTools Performance tab
- `[Violation] 'requestAnimationFrame' handler took Nms` warnings
- Frame drops precisely when particles die or projectiles expire
- Stutter that gets worse the longer the session runs
- 30–50ms GC pause gaps in the Performance flame chart

**Phase to address:**
Phase implementing the game loop and rendering — establish zero-allocation discipline before particle effects or trail systems are added. Add lint rules disallowing `new` in hot paths during code review.

---

### Pitfall 2: Touch Scroll During Gameplay (Missing `passive: false`)

**What goes wrong:**
When the player drags to pull the bow, the page scrolls, the game jumps out of view, and the shot is ruined. On iOS especially, scroll inertia continues after finger lift, taking several seconds to settle.

**Why it happens:**
- Modern browsers default touch listeners to `passive: true` for performance — calls to `e.preventDefault()` inside passive listeners are silently ignored
- Developers add `touchmove` listeners on the document but forget `{ passive: false }` in the third parameter
- CSS `touch-action: none` alone may not prevent all browser gestures (pull-to-refresh, swipe-navigation on iOS)
- `overflow: hidden` on `<body>` doesn't prevent touch scrolling on iOS Safari (the viewport still bounces)

**How to avoid:**
Apply ALL three layers of prevention:
1. **CSS**: `html, body { overflow: hidden; touch-action: none; position: fixed; width: 100%; height: 100%; }`
2. **Meta**: `<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover, user-scalable=no">`
3. **JS**: `canvas.addEventListener('touchmove', (e) => e.preventDefault(), { passive: false })` on the canvas element itself

For the bow drag gesture specifically:
- On `touchstart` on the canvas, begin tracking the pull
- On `touchmove`, update pull distance AND call `e.preventDefault()`
- On `touchend`, fire the arrow
- Handle `touchcancel` — reset the bow state (important on iOS when system UI takes over)

**Warning signs:**
- Page bounces vertically when testing drag on a real iPhone
- Chrome DevTools mobile emulation shows scrollbars during touch simulation
- Arrow release position is offset from where the user touched
- Double-tap zooms the page instead of starting a new shot

**Phase to address:**
Phase implementing InputHandler. This must be baked into the input architecture from day one — retrofitting is painful.

---

### Pitfall 3: Canvas Blurry on Retina/HiDPI Displays

**What goes wrong:**
On iPhones (DPR 2–3), high-end Android phones (DPR 2.5–4), and Retina MacBooks, the canvas appears fuzzy or pixelated. Lines and text look soft. The game looks unpolished despite being "correct."

**Why it happens:**
Canvas has TWO sizes: the CSS display size (logical pixels) and the internal bitmap resolution (physical pixels). By default, a `<canvas>` element's internal resolution matches its CSS size. On a 2x display, a 375×667 CSS canvas only has 375×667 physical pixels — the browser stretches it to 750×1334 screen pixels, causing blur.

The fix requires scaling the internal bitmap by `window.devicePixelRatio` and then scaling the drawing context to compensate.

**How to avoid:**
```typescript
function resizeCanvas(canvas: HTMLCanvasElement, ctx: CanvasRenderingContext2D) {
  const dpr = window.devicePixelRatio || 1;
  const width = canvas.clientWidth;
  const height = canvas.clientHeight;

  // Only resize if dimensions actually changed (prevents context reset churn)
  if (canvas.width !== Math.floor(width * dpr) || canvas.height !== Math.floor(height * dpr)) {
    canvas.width = Math.floor(width * dpr);
    canvas.height = Math.floor(height * dpr);
    // Reset transform first (scale is cumulative!)
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    ctx.imageSmoothingEnabled = true; // Re-apply after resize (context state resets!)
  }
}
```

**Key gotchas:**
- `ctx.scale(dpr, dpr)` is CUMULATIVE — call it once, not on every resize. Use `setTransform()` instead to reset
- Setting `canvas.width` or `canvas.height` CLEARS the canvas AND resets ALL context state (transform, fillStyle, strokeStyle, globalAlpha, imageSmoothingEnabled, etc.) — re-apply these after resize
- On mobile with DPR >= 3, consider capping effective DPR to 2 for performance: `const effectiveDpr = Math.min(dpr, 2)`
- Listen for `window.matchMedia('(resolution: Xdppx)')` changes — DPR changes when dragging between monitors

**Warning signs:**
- Text is noticeably softer than HTML/DOM text at the same CSS size
- 1px lines appear 2px wide on iPhones
- Game looks crisp on desktop but blurry on mobile

**Phase to address:**
Phase implementing Renderer and canvas setup. Must be present from the first frame drawn.

---

### Pitfall 4: Fixed-Update Game Loop Without Delta Clamping

**What goes wrong:**
When the player switches tabs and returns after 30 seconds, the game loop receives a single `requestAnimationFrame` call with a 30,000ms delta. The accumulated time causes the update loop to run 1,800 "frames" instantly — the arrow teleports off-screen, the target flies past, or the game state corrupts. On 120Hz ProMotion iPhones, game physics runs at double speed because rAF fires every 8.33ms instead of 16.67ms.

**Why it happens:**
- Developers assume rAF fires at exactly 60fps and increment positions by fixed amounts per frame
- No `deltaTime` calculation at all (frame-dependent physics)
- `deltaTime` is calculated but not clamped, allowing unbounded accumulator growth
- Physics and rendering are in the same function with no separation of timestep concerns

**How to avoid:**
Implement a fixed-timestep game loop with accumulator:

```typescript
const FIXED_DT = 1 / 60;      // 60 physics updates per second
const MAX_DELTA = 0.25;        // Cap to prevent spiral of death (250ms)

let accumulator = 0;
let lastTime = performance.now();

function gameLoop(now: DOMHighResTimeStamp) {
  const frameDelta = Math.min((now - lastTime) / 1000, MAX_DELTA);
  lastTime = now;
  accumulator += frameDelta;

  while (accumulator >= FIXED_DT) {
    updateGameState(FIXED_DT);  // Physics, collision, arrow flight
    accumulator -= FIXED_DT;
  }

  const alpha = accumulator / FIXED_DT;  // Interpolation factor
  render(alpha);                         // Draw current state

  requestAnimationFrame(gameLoop);
}
```

**Critical details:**
- Separate `update()` (game logic) from `render()` (drawing) — they run at different rates
- The `MAX_DELTA` cap (0.25s = 250ms) prevents physics explosion when returning from background
- On 120Hz displays, update still runs at 60Hz but render runs at 120Hz — use `alpha` to interpolate rendered positions
- Read `document.visibilityState` — pause the accumulator when the tab is hidden, don't just rely on rAF pausing (Safari throttles rAF to 1fps in background tabs)

**Warning signs:**
- Arrow flies at different speeds on iPhone vs Android
- Game "fast-forwards" when switching back to the tab
- Sporadic teleportation after phone call interruptions
- Physics feels different on 120Hz iPad Pro vs 60Hz iPhone

**Phase to address:**
Phase implementing GameLoop. This is the architectural foundation — cannot be changed later without rewriting the entire loop.

---

### Pitfall 5: iOS Safari Viewport Height Changes (Address Bar + Safe Areas)

**What goes wrong:**
The game canvas jumps or leaves a blank strip at the bottom when Safari's address bar collapses/expands during scrolling. On notched iPhones, the target is partially hidden behind the Dynamic Island or the home indicator bar. `100vh` in CSS includes the area behind the address bar, causing content to be cut off.

**Why it happens:**
- iOS Safari dynamically resizes `window.innerHeight` when the address bar shows/hides
- `100vh` in CSS refers to the initial viewport, not the visible viewport
- Without `viewport-fit=cover`, iOS insets content from the notch/home indicator automatically, shrinking the viewport
- `window.innerHeight` changes can happen mid-game when the user taps near the bottom edge

**How to avoid:**
1. Add to viewport meta tag: `viewport-fit=cover`
2. Use dynamic CSS height with JS fallback:
   ```css
   #gameCanvas {
     height: 100vh;  /* fallback */
     height: calc(var(--vh, 1vh) * 100);
   }
   ```
   ```typescript
   function setViewportHeight() {
     const vh = window.innerHeight * 0.01;
     document.documentElement.style.setProperty('--vh', `${vh}px`);
   }
   window.addEventListener('resize', setViewportHeight);
   window.addEventListener('orientationchange', () => setTimeout(setViewportHeight, 100));
   setViewportHeight();
   ```
3. Add safe area padding for notched devices:
   ```css
   body {
     padding-top: env(safe-area-inset-top);
     padding-bottom: env(safe-area-inset-bottom);
     padding-left: env(safe-area-inset-left);
     padding-right: env(safe-area-inset-right);
   }
   ```
4. For the game canvas specifically, use `position: fixed` and size via JS:
   ```typescript
   canvas.style.position = 'fixed';
   canvas.style.top = '0';
   canvas.style.left = '0';
   function resize() {
     canvas.style.width = window.innerWidth + 'px';
     canvas.style.height = window.innerHeight + 'px';
   }
   ```

**Warning signs:**
- Black bar at bottom of canvas on iPhone X+
- Game extends below the visible area on iOS Safari
- Canvas jumps when user taps near bottom of screen
- Target/UI elements hidden behind the home indicator
- Landscape mode cuts off gameplay elements

**Phase to address:**
Phase implementing canvas setup and viewport initialization. Must be in the first playable build.

---

### Pitfall 6: LocalStorage Throws in Safari Private Browsing Mode

**What goes wrong:**
High scores never persist. `localStorage.setItem('highScore', ...)` silently throws a `QuotaExceededError` (DOM Exception 22) in Safari private/incognito mode. The app appears to work — reads return null, writes throw — but the game doesn't crash visibly. After the private tab closes, all progress is lost. Players don't know they're in private mode.

**Why it happens:**
- Safari private browsing assigns quota = 0 to localStorage
- `setItem()` always throws, but reading `localStorage` properties returns null without error
- The standard `if (window.localStorage)` check passes, giving false confidence
- Older Safari versions (pre-iOS 15) silently fail writes
- In-app browsers (Instagram, Discord, Twitter) have various storage restrictions

**How to avoid:**
Wrap ALL localStorage operations in try-catch and provide in-memory fallback:

```typescript
const storage = {
  getItem(key: string): string | null {
    try {
      return localStorage.getItem(key);
    } catch {
      console.warn(`localStorage read failed for "${key}"`);
      return null;
    }
  },
  setItem(key: string, value: string): boolean {
    try {
      localStorage.setItem(key, value);
      return true;
    } catch {
      console.warn(`localStorage write failed for "${key}" — private mode?`);
      inMemoryFallback.set(key, value);
      return false;
    }
  },
  removeItem(key: string): void {
    try {
      localStorage.removeItem(key);
    } catch {
      // silent fail
    }
  }
};
```

**Additional protections:**
- In-memory fallback for the current session so the game works regardless
- After 2+ write failures, show a discreet notification: "High scores won't save in private mode"
- On `visibilitychange`, attempt an emergency save (mobile browsers may evict tab)
- Consider requesting persistent storage: `navigator.storage.persist()` (may prompt user)
- Keep stored data minimal (< 100KB) — high score is just a number

**Warning signs:**
- High score resets every session (most common report)
- `QuotaExceededError` in Safari console
- Write failures only on iOS, works on desktop
- User reports from iPhone users about score loss

**Phase to address:**
Phase implementing storage module. Design the wrapper layer before any feature depends on persistence.

---

### Pitfall 7: Not Separating Update from Render (Game Loop Architecture)

**What goes wrong:**
Game physics runs at the display refresh rate. On a 120Hz iPad Pro, the arrow moves at double speed. On a 60Hz budget Android, it moves at normal speed. The collision system misses the target because the arrow skipped past it between frames. Frame-rate-dependent physics creates inconsistent gameplay across devices.

**Why it happens:**
- The game loop calls a single `update()` function that both computes physics and draws
- No concept of "fixed timestep" — every frame advances by a fixed increment regardless of elapsed time
- The natural assumption is "60fps = 60 updates/second" which breaks at 120Hz or 30fps

**How to avoid:**
The fixed-timestep accumulator pattern (detailed in Pitfall 4) is the standard. Key rules:
1. `update(FIXED_DT)` runs game logic at fixed 1/60s intervals — always deterministic
2. `render(alpha)` draws the current state — runs every rAF callback
3. All user input is sampled once per rAF, before the accumulator loop
4. Arrow trajectory uses time-based physics, not frame-count-based

For the archery game specifically:
- Pull distance is measured in CSS pixels, sampled from touch position
- Arrow velocity is computed from pull distance (pixels → physics units)
- Gravity is a constant applied per-unit-time, not per-frame
- Collision detection happens at the target plane using the arrow's interpolated position

**Warning signs:**
- Arrow speed differs between iPhone and Android
- Rapid tab switching causes unpredictable arrow behavior
- Jittery animation when rAF fires at irregular intervals
- Same build runs at different difficulty on different devices

**Phase to address:**
Phase implementing GameLoop. Architectural decision that affects every subsequent phase.

---

### Pitfall 8: Canvas Context State Reset on Resize

**What goes wrong:**
After orientation change or window resize, the canvas clears, the drawing transform resets, `imageSmoothingEnabled` reverts to default, fill styles disappear. The game renders one frame at the wrong scale or with wrong colors before correcting — a visible flash.

**Why it happens:**
Per the Canvas specification, setting `canvas.width` or `canvas.height` automatically:
1. Clears the canvas bitmap
2. Resets the transform to identity
3. Resets `globalAlpha`, `globalCompositeOperation`
4. Resets `strokeStyle`, `fillStyle`, `lineWidth`, `lineCap`, `lineJoin`
5. Resets `shadowBlur`, `shadowColor`, `shadowOffsetX`, `shadowOffsetY`
6. Resets `font`, `textAlign`, `textBaseline`
7. Resets `imageSmoothingEnabled` to `true`

**How to avoid:**
- Only resize when dimensions actually change (guard check)
- Bundle all state re-initialization after resize into a single function
- Keep your own rendering state object to replay onto context

```typescript
function ensureCanvasSize(width: number, height: number, dpr: number) {
  const w = Math.floor(width * dpr);
  const h = Math.floor(height * dpr);
  if (canvas.width === w && canvas.height === h) return false; // Skip if unchanged

  canvas.width = w;
  canvas.height = h;
  return true; // Signal caller to reapply state
}

// In the render loop:
if (resized) {
  ctx.setTransform(effectiveDpr, 0, 0, effectiveDpr, 0, 0);
  ctx.imageSmoothingEnabled = true;
  // Reapply any custom styles...
}
```

**Warning signs:**
- One white frame on orientation change
- Arrow drawn at wrong position after screen rotation
- Text renders wrong size for one frame after resize
- One frame of jagged edges after resize (imageSmoothingEnabled reset)

**Phase to address:**
Phase implementing Renderer and resize handling.

---

### Pitfall 9: `drawImage` Scaling Every Frame Instead of Caching

**What goes wrong:**
Every frame, the renderer calls `ctx.drawImage(arrowSprite, x, y, 64, 64)` — scaling the source image from its natural size. On mobile devices, this per-frame GPU work adds 3-8ms of draw time per call. With multiple sprites (arrow, bow, target, background), the budget is consumed by scaling work.

**Why it happens:**
- Loading one sprite and scaling it on every frame is the simplest code
- Developers don't realize `drawImage` with different dest sizes forces GPU to scale each time
- Mobile GPUs are especially sensitive to unbatched, varying-size drawImage calls
- No offscreen canvas caching for static elements (target, background)

**How to avoid:**
- Pre-render static elements (target, background) onto offscreen canvases once at the correct display size
- Cache scaled sprites at the required dimensions after load, not per-frame
- Use atlas batching: group drawImage calls by source texture to minimize GPU state changes

```typescript
// Pre-render target to offscreen canvas (once)
const targetCanvas = document.createElement('canvas');
targetCanvas.width = targetWidth;
targetCanvas.height = targetHeight;
const targetCtx = targetCanvas.getContext('2d');
// Draw target rings, colors onto offscreen canvas...
// In render loop: just one drawImage call
ctx.drawImage(targetCanvas, x, y);
```

- For the background (sky, ground), these static elements should be drawn once and never redrawn unless resized
- Use CSS `background-image` for purely decorative static backgrounds instead of drawing them on canvas

**Warning signs:**
- 60fps on desktop, drops to 30fps on mobile
- Performance profiler shows `drawImage` consuming >5ms per frame
- Battery heats phone noticeably during gameplay
- Frame time increases proportionally to number of sprites on screen

**Phase to address:**
Phase implementing Renderer and asset loading. Establish caching patterns before adding complex visuals.

---

### Pitfall 10: Getting Touch Canvas Coordinates Wrong

**What goes wrong:**
The arrow fires offset from where the user touched. On higher-DPR devices, the pull distance is half what it should be. On mobile, the touch position doesn't match the drawn bow position.

**Why it happens:**
- `e.touches[0].pageX` gives document-relative coordinates, not canvas-relative
- The canvas may be offset from the document by CSS positioning
- `clientX/clientY` is viewport-relative, while the canvas may not start at (0,0)
- DPR scaling means physical touch coordinates need to be divided by DPR to get canvas-logical coordinates

**How to avoid:**
Use `canvas.getBoundingClientRect()` to convert coordinates:

```typescript
function getTouchPos(canvas: HTMLCanvasElement, touch: Touch): { x: number, y: number } {
  const rect = canvas.getBoundingClientRect();
  return {
    x: touch.clientX - rect.left,
    y: touch.clientY - rect.top
    // These are now in CSS-pixel coordinates matching the drawing space
  };
}
```

No DPR division needed here — the canvas draw operations are scaled by DPR, but touch coordinates in CSS pixels already match the logical drawing space. Only divide by DPR if you need to read back physical pixels from the canvas.

**Additional gotchas:**
- `getBoundingClientRect()` triggers layout — cache it at the start of the frame, don't call per-touch
- Finger size: the actual touch point is larger than a mouse cursor — add a dead zone or tolerance
- `touchcancel` must reset the bow state (iOS fires this when a system notification appears)

**Warning signs:**
- Arrow fires above or to the left of finger position
- Pull distance feels different on portrait vs landscape
- On iOS, the first touch position is offset after keyboard dismissal
- Touches near canvas edges produce wrong positions

**Phase to address:**
Phase implementing InputHandler. Test on real devices immediately.

---

### Pitfall 11: Cumulative `ctx.scale()` and Duplicate Resize Calls

**What goes wrong:**
After the second resize event (orientation change, keyboard open/close), the canvas is scaled by DPR² — everything draws at 4x size on a 2x display. Only a quarter of the game is visible; the rest is offscreen.

**Why it happens:**
`ctx.scale(dpr, dpr)` is CUMULATIVE. Calling it again on the next resize without resetting doubles the scaling factor. The same issue affects any state-setting calls inside resize handlers that fire multiple times.

**How to avoid:**
Use `setTransform()` which resets the full transform matrix:

```typescript
// WRONG — cumulative:
ctx.scale(dpr, dpr);  // First call: 2x
ctx.scale(dpr, dpr);  // Second call: 4x — BUG!

// CORRECT — resets first:
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);  // Always exactly 2x
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);  // Still 2x — idempotent
```

Or guard against unnecessary resizes:
```typescript
let lastCanvasWidth = 0;
let lastCanvasHeight = 0;

function resizeIfChanged() {
  const dpr = window.devicePixelRatio || 1;
  const w = Math.floor(window.innerWidth * dpr);
  const h = Math.floor(window.innerHeight * dpr);
  if (canvas.width !== w || canvas.height !== h) {
    canvas.width = w;
    canvas.height = h;
    lastCanvasWidth = w;
    lastCanvasHeight = h;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    ctx.imageSmoothingEnabled = true;
  }
}
```

**Warning signs:**
- After rotating the phone, only top-left quadrant of the game is visible
- Debug overlay shows scale > 1
- UI elements are enormous after second orientation change
- Works on first load, breaks after resize

**Phase to address:**
Phase implementing canvas resize handling.

---

### Pitfall 12: No Background Tab Handling

**What goes wrong:**
When the user switches to another tab and returns, the game arrow has already flown, the round is over, or the game state is corrupted. The audio context (when added later) fails because Chrome suspended it while the tab was hidden.

**Why it happens:**
- `requestAnimationFrame` is throttled to ~1fps in background tabs (browser battery saving)
- The accumulator grows unboundedly without visibility-aware pausing
- Audio context is suspended by browser policy when tab is hidden
- Timer-based events (delays, round timers) continue in background, desynchronizing from visual state

**How to avoid:**
```typescript
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // Tab hidden — pause the game
    isPaused = true;
    // Suspend any audio context
    audioContext?.suspend();
  } else {
    // Tab visible again — resume
    isPaused = false;
    // Reset timing to avoid delta spike
    lastTime = performance.now();
    accumulator = 0;
    // Resume audio
    audioContext?.resume();
  }
});
```

Also handle:
- `pagehide` / `beforeunload` events for saving high score on tab close
- `touchcancel` for when iOS system UI overlays the game (incoming call, control center)
- Round timer: use `Date.now()` or `performance.now()` relative to game start, not frame count

**Warning signs:**
- Arrow has already flown when returning to the game
- "Game over" screen shows immediately on tab refocus
- Round ended prematurely
- Audio doesn't work after returning to the tab

**Phase to address:**
Phase implementing GameLoop. Add visibility handling alongside the main loop.

---

### Pitfall 13: Not Handling `touchcancel` (iOS System Overlays)

**What goes wrong:**
When the user drags to aim and a system notification appears (incoming call, alarm, low battery), the touch is cancelled. The bow stays in the "pulled" position. The next touch doesn't reset it. The game is stuck in the `aiming` state.

**Why it happens:**
- `touchend` isn't always fired — iOS fires `touchcancel` for system interruptions
- The game only handles `touchend` and never listens for `touchcancel`
- The game state machine doesn't have a "cancel everything" transition

**How to avoid:**
Always handle `touchcancel` alongside `touchend`:

```typescript
canvas.addEventListener('touchend', (e) => {
  e.preventDefault();
  if (gameState === 'aiming') {
    fireArrow(currentPullDistance, currentAimAngle);
    gameState = 'flying';
  }
  isPulling = false;
}, { passive: false });

canvas.addEventListener('touchcancel', (e) => {
  // System cancelled the touch — reset bow without firing
  isPulling = false;
  currentPullDistance = 0;
  currentAimAngle = 0;
  gameState = 'idle';
  // Optionally render a "cancelled" visual feedback
}, { passive: false });
```

**Warning signs:**
- Bow stays pulled after receiving a phone call
- First touch after alarm doesn't register
- Arrow fires spontaneously after switching back to tab
- Touch state gets "stuck" on iOS but not Android

**Phase to address:**
Phase implementing InputHandler. Must be part of the initial touch handling design.

---

### Pitfall 14: Improper `imageSmoothingEnabled` for Arrow/Assets

**What goes wrong:**
Sprites look blurry or have white 1-pixel edges on mobile. The arrow, bow, or target ring edges appear smeared. The pixel-art style is lost.

**Why it happens:**
- Canvas defaults to bilinear image smoothing (`imageSmoothingEnabled = true`)
- Bilinear filtering blurs edges when sprites are scaled or at sub-pixel positions
- On Retina displays, the DPR scaling + bilinear smoothing = double blurring
- Sprite edges have partial alpha pixels that get averaged with neighboring transparent areas

**How to avoid:**
```typescript
// After setting up canvas, explicitly set:
ctx.imageSmoothingEnabled = true;  // For smooth, non-pixel-art games
// OR for pixel art:
ctx.imageSmoothingEnabled = false; // Crisp edges
```

For this archery game (not pixel art):
- `imageSmoothingEnabled = true` is correct for smooth vector-like rendering
- But re-apply it after EVERY canvas resize (context state resets!)
- Ensure sprite assets have clean alpha edges (no anti-aliased fringing at sprite boundaries in the image file)
- Keep sprite positions at whole pixels: `Math.round(x)`, `Math.round(y)` to avoid sub-pixel blur

**Warning signs:**
- White or dark halos around arrow/bow sprites
- Flickering at the edges of moving objects
- Sprites look fine in PNG viewer but blurry on canvas
- Diagonal lines appear jagged or stepped

**Phase to address:**
Phase implementing Renderer and asset loading.

---

### Pitfall 15: Not Capping Frame Budget on Low-End Devices

**What goes wrong:**
On a Pixel 6 or Samsung A-series, the game runs at 60fps for the first few rounds, then the phone heats up, the CPU throttles, and frame rate drops to 20-30fps. The game becomes sluggish and unresponsive.

**Why it happens:**
- The game targets 60fps unconditionally, with no frame budget monitoring
- CANVAS 2D rendering is CPU-bound — sustained load triggers thermal throttling
- Mobile CPUs have aggressive thermal management; after 30-60 seconds at full load, they downclock
- Frame time budget (16.67ms) is exceeded but no adaptation mechanism exists

**How to avoid:**
- Monitor frame time every second — if consistently over 14ms, degrade quality:
  - Reduce particle effect complexity
  - Lower effective DPR (e.g., cap at 1.5 instead of 2)
  - Skip non-essential visual effects (arrow trail, floating score text at reduced rate)
  - Reduce draw calls by merging static objects
- Use a quality score that auto-adjusts:
  ```typescript
  let qualityLevel = 1.0; // 1.0 = full quality, 0.5 = reduced

  function checkPerformance() {
    if (frameTimeMovingAvg > 14) {
      qualityLevel = Math.max(0.5, qualityLevel - 0.1);
      // Apply: reduce particles, lower DPR, skip certain effects
    } else if (frameTimeMovingAvg < 8 && qualityLevel < 1.0) {
      qualityLevel = Math.min(1.0, qualityLevel + 0.1);
    }
  }
  ```
- Start at a reasonable resolution: a 480×270 logical game canvas scaled up looks much better than a 1920×1080 one running at 20fps

**Warning signs:**
- Phone gets warm after 2-3 rounds
- FPS drops below 30 after sustained play
- Chrome DevTools shows frame times climbing over time (thermal throttling)
- Works for first 30 seconds, degrades afterward

**Phase to address:**
Phase implementing GameLoop and Renderer. The adaptive quality system should be designed alongside the renderer, not retrofitted.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| **`new` objects in update loop** | Fast to write, readable code | GC stutter at 60fps — requires full refactor to pool pattern | Never — causes observable jank |
| **`setTimeout`/`setInterval` for game loop** | Simple to understand | Jitter, no VSync, continues in background — frame skipping | Never for the rendering loop |
| **One monolithic update+render function** | No architecture overhead | Impossible to separate physics timestep from draw rate, coupling prevents debugging | Only in prototypes < 100 lines |
| **`window.innerHeight` without resize listener** | One-line viewport sizing | Broken on iOS Safari (address bar resize) | Never for mobile — always include resize handling |
| **Hardcoded canvas CSS size** | Looks correct on dev machine | Blurry on every other device; wrong aspect ratio on tablets | Only for internal dev tools, never shipped |
| **No try-catch on localStorage** | Saves 3 lines of code | Silent failure in Safari private mode; scores lost silently | Never — production data loss |
| **`Math.random()` for particle variation** | Simple, works | Non-deterministic — hard to debug collision bugs | Only for purely cosmetic effects |
| **Synchronous audio on touch event** | Instant feedback | Blocked by browser autoplay policy; breaks on iOS | Never — use Web Audio API with resume() |
| **One `touchmove` listener on document** | Catches all touch events | Can't selectively `preventDefault()`; may block native scrolling on other UI | Acceptable only if entire page is a game canvas |
| **Setting canvas size on every frame** | Easy to implement | Context state reset every frame; all transforms, styles lost | Never — guard with equality check |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| **Per-frame `getImageData()`** | Massive frame drops, 500+ms frame times | Never use `getImageData()` in render loop — it forces GPU→CPU readback | Any usage in the render loop |
| **`for...of` on large arrays** | GC allocation per iteration | Prefer indexed `for` loops in hot update paths | ~300+ entities in a 60fps loop |
| **Recreating gradient/pattern every frame** | GPU state churn, draw call overhead | Cache gradient objects, reuse across frames | ~50+ objects per frame |
| **Setting `shadowBlur` per call** | 5-10x slowdown on complex shapes | Use offscreen canvas for shadow effects, draw image | Any use in 60fps game loop |
| **DOM UI updates every frame** | Layout thrashing — frame spikes > 20ms | Use DOM overlay only for infrequent UI changes; update via CSS classes not inline styles | 1+ UI update per frame |
| **Loading all assets at game start** | Long initial load, memory pressure on low-end | Load by screen/scene; preload next scene assets during gameplay | >2MB total assets on 1GB RAM devices |
| **String concatenation in render loop** | GC allocation per concat — builds up | Pre-compute display strings, use template literals sparingly | Every frame with 5+ concatenations |
| **Re-creating offscreen canvases on resize** | Memory spike, frame drop on rotation | Pre-create and just re-draw content | Every orientation change |
| **Global `touchmove` listener with no guard** | All touch events processed even when game is paused | Check game state before processing | Any pause/menu screen |

---

## UX Pitfalls

Common user experience mistakes in this domain.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| **No visual feedback on touch start** | Player unsure if game registered the touch | Show bow flexing/pulsing on `touchstart` — even before pull distance registers |
| **Drag-to-pull requires too much distance** | Player can't reach full power before running out of screen | Scale pull distance so full power = ~60% of screen height; dead zone before any power registers |
| **No dead zone at start of pull** | Accidental shots from just placing finger on screen | Ignore pull distances < 10 CSS pixels; require deliberate movement |
| **Arrow trail creates visual noise** | Can't see where the arrow is going | Fade trail opacity from 1 to 0; limit trail segments to 8-12 |
| **No feedback on miss** | Player doesn't know where the arrow went | Show arrow stuck in ground/sky briefly before fade-out; or show a "miss" indicator on target edge |
| **Round ends without warning** | "Game over" feels sudden | Show arrow count remaining; subtle effect when last arrow lands |
| **High score display is tiny** | No sense of achievement | Big, animated number on results screen; use DOM overlay for text styling flexibility |
| **Orientation lock not requested** | Player accidentally rotates mid-round | Request landscape lock: `screen.orientation.lock('landscape')` — but provide fallback for portrait players |
| **No haptic feedback on arrow release** | Feels disconnected on mobile | `navigator.vibrate(15)` on release (Android only; gracefully ignore on iOS) |
| **First launch has no "how to play"** | Player taps aimlessly | Self-teaching via animated instruction overlay (tap to dismiss) showing the drag gesture |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Touch input handler:** `touchcancel` event not handled — bow state gets stuck on iOS interruption
- [ ] **Canvas rendering:** `imageSmoothingEnabled` not re-applied after resize — sudden blurriness after orientation change
- [ ] **Game loop:** Delta not clamped — physics explosion when tab returns from background
- [ ] **Storage:** `localStorage.setItem` not wrapped in try-catch — scores silently lost in Safari private mode
- [ ] **Viewport:** `viewport-fit=cover` missing — content hidden behind notch/home indicator on iPhone X+
- [ ] **Resize:** Canvas context state not re-initialized after resize — one-frame visual glitch on rotation
- [ ] **Performance:** No frame time monitoring — game degrades silently on low-end devices
- [ ] **Background tab:** Game loop not paused on `visibilitychange` — arrow teleports when returning
- [ ] **Touch coordinates:** Coordinates not converted via `getBoundingClientRect()` — offset on non-fullscreen canvases
- [ ] **DevicePixelRatio:** Canvas not scaled by DPR — blurry on Retina displays
- [ ] **Safe areas:** No `env(safe-area-inset-*)` padding — home indicator overlaps UI on modern phones
- [ ] **Sprite edges:** `drawImage` at non-integer positions causing sub-pixel blur — shimmering movement

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| GC stutter from allocations | HIGH — requires pooling refactor | 1. Profile to find allocation hot spots. 2. Replace `new` with Pool. 3. Replace `.filter()/.map()` with loops. 4. Prewarm pools at load. Could take 2-3 days. |
| Background tab delta spike | MEDIUM | Add `Math.min(delta, 0.25)` cap in the loop. Add `visibilitychange` listener. Reset `lastTime` on resume. ~2 hours. |
| Blurry canvas on Retina | LOW | Add DPR scaling. ~30 minutes if architecture is clean. 1-2 days if rendering is tightly coupled. |
| localStorage private mode | LOW | Wrap all storage in try-catch. ~1 hour. |
| Touch offset on mobile | LOW-MEDIUM | Fix coordinate conversion. ~1-2 hours if input architecture is separate. |
| Cumulative `ctx.scale()` | LOW | Replace with `setTransform()`. ~15 minutes. |
| Missing `touchcancel` | LOW | Add event listener. ~30 minutes. |
| Viewport not accounting for address bar | MEDIUM | Add `--vh` variable + resize listener. ~2 hours including testing. |
| Canvas context state loss on resize | MEDIUM | Create state-bundle function. ~3-4 hours to audit all state. |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Per-frame GC allocations (P1) | GameLoop + Renderer phases | Chrome DevTools memory timeline shows flat line, no sawtooth |
| Touch scroll (P2) | InputHandler phase | Real device test: drag across canvas, page does not scroll |
| Blurry canvas (P3) | Canvas setup (first phase) | Screenshot on iPhone at 100% zoom, text/shapes are crisp |
| Delta not clamped (P4) | GameLoop phase | Set Chrome throttle to "Slow 3G", switch tabs for 10s, return — no teleport |
| iOS viewport height (P5) | Canvas setup phase | Test on iOS Safari with address bar visible/collapsed |
| localStorage private mode (P6) | Storage module phase | Open in Safari private mode, scores save silently |
| Update/render not separated (P7) | GameLoop phase | 120Hz iPad runs physics same speed as 60Hz Android |
| Context reset on resize (P8) | Renderer phase | Rotate device multiple times, no visual glitch |
| `drawImage` scaling (P9) | Renderer + Assets phase | Profiler shows drawImage < 2ms per frame |
| Touch coordinate conversion (P10) | InputHandler phase | Arrow fires exactly where finger was at release |
| Cumulative `ctx.scale()` (P11) | Canvas setup phase | Resize twice, game renders at correct scale |
| Background tab handling (P12) | GameLoop phase | Switch tabs for 30s, return — game state intact |
| Missing `touchcancel` (P13) | InputHandler phase | Simulate iOS notification while dragging — bow resets |
| `imageSmoothingEnabled` (P14) | Renderer phase | After orientation change, sprites aren't blurry |
| Performance capping (P15) | GameLoop + Renderer | Pixel 6 stays at 60fps after 5 minutes of gameplay |

---

## Sources

- **Zero-allocation TypeScript game loops** (DEV Community, 2026-05-26) — Real-device benchmarks across 3 Android devices showing GC impact
- **How HTML5 Canvas Performance Actually Works** (Hex Hour, 2026-05-04) — GPU-accelerated Canvas pipeline in modern browsers
- **Optimizing Canvas Performance** (BSWEN, 2026-02-21) — Offscreen canvas caching, DPR handling, alpha:false, desynchronized
- **Mobile-friendly web games** (Cinevva) — Touch event handling, fullscreen canvas, CSS prevention
- **MDN: Viewport meta tag** — viewport-fit=cover, interactive-widget, visual viewport
- **MDN: Optimizing canvas** — drawImage scaling, CSS transforms, DPR handling
- **MDN: Storage quotas and eviction criteria** — 5-10MB localStorage limit, private browsing behavior
- **Polypane: Using safe-area-inset** — env(safe-area-inset-*) usage, viewport-fit=cover
- **Game Save Best Practices** (Bugnet, 2026-03-31) — localStorage pitfalls, persistent storage API
- **TrackJS: Failed to execute 'setItem' on 'Storage'** — Safari private mode workaround
- **WebKit Bug #157010** — localStorage quota 0 in private mode
- **Web.dev: High DPI Canvas** — devicePixelRatio setup pattern
- **Web.dev: Static memory JavaScript with Object Pools** — GC patterns, pool implementation
- **Dev Blog: Optimize your game loop for 60fps** (2025-06-25) — Fixed timestep, delta clamping
- **Aleksandr Hovhannisyan: Performant Game Loops** (2024-12-29) — MAX_FPS clamping, input handling in rAF
- **How I optimized my Phaser 3 action game** (Medium, 2025-02-23) — Canvas vs WebGL real-device comparison (30% faster on Canvas)
- **Chrome DevRel: Speed Rendering** — Frame budget breakdown, jank causes
- **Cinevva: Responsive game canvas** — DPR, letterboxing, integer scaling for pixel art
- **Cinevva: Game input handling** — Touch event selection, passive:false
- **MDN: Crisp pixel art look** — image-rendering CSS, sub-pixel shimmer fix

---
*Pitfalls research for: Archery mobile browser game*
*Researched: 2026-06-08*
