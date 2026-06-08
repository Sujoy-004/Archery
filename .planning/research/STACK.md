# Stack Research

**Domain:** Mobile-first HTML5 Canvas 2D arcade archery game
**Researched:** 2026-06-08
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| TypeScript | 6.0.x | Static typing for game logic, state, and rendering | 2025/2026 standard for browser game projects — eliminates entire categories of runtime bugs (null refs, impossible states, magic strings). All major game templates (insertcoin, Phaser, Excalibur) use TS-first. TS 6.0 ships improved type inference for discriminated unions used heavily in game state machines. |
| Vite | 8.0.x | Build tool and dev server | Industry standard replacement for Webpack. Sub-300ms HMR, native ESM dev server, esbuild transpilation. Vite 8 ships native module chunking and improved worker support. Zero-config for vanilla TS + Canvas projects. No game-specific plugin needed. |
| HTML5 Canvas 2D | Native API | Game rendering | The right choice for a simple 2D archery game. Canvas 2D avoids the ~800KB+ bundle cost of WebGL engines (PixiJS, Phaser) for a game with ~10 draw calls per frame. Canvas 2D has universal mobile browser support, zero dependency burden, and straightforward pixel-based drawing for circles, lines, images, and text that this game needs. |
| requestAnimationFrame | Native API | Game loop timing | Standard for all browser games. Browser-synced vsync, automatic pause when tab is hidden (mobile battery saving), frame-accurate timing. Use delta-time normalized to 60 FPS target for frame-rate-independent physics. No library needed — ~20 lines of code. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| vite-plugin-pwa | 1.3.0 | PWA service worker generation | For offline support after first load. Wraps Workbox 7, generates sw.js + manifest.webmanifest at build time. Only needed because AUTH-09 requires offline-capable. |
| @types/node | (bundled with TS) | Node.js type definitions | Required for Vite config `import.meta.env` and path resolution. Dev dependency only. |

**No physics library, no input library, no game engine, no state management library.** This project's complexity (~500-800 lines game logic) does not justify external runtime dependencies. Every required feature (trajectory physics, pointer input, game loop, collision detection) is <50 lines of straightforward code.

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| VS Code | Editor | TypeScript built-in, no extensions needed for game dev |
| Chrome DevTools | Performance profiling, canvas debugging, mobile emulation | Use `More Tools > Rendering > FPS meter` to verify 60 FPS on target devices. Paint flashing for canvas redraw verification. |
| npm | Package manager | Ships with Node. Use `npm create vite@latest -- --template vanilla-ts` to scaffold. |

## Installation

```bash
# Create project
npm create vite@latest archery-game -- --template vanilla-ts
cd archery-game

# Core (only PWA plugin as runtime dependency)
npm install

# PWA support
npm install -D vite-plugin-pwa

# Dev
npm run dev
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Vanilla Canvas 2D | PixiJS 8.x | When you need >50 drawable objects per frame with batching, or sprite sheets, or WebGL filters. Overkill for this archery game (~5-10 objects on screen). |
| Vanilla Canvas 2D | Phaser 4.x | When you need a complete scene manager, physics engine (arcade/matter), audio system, and plugin ecosystem. The ~450KB bundle cost is unjustified for a single-screen archery game. |
| Vanilla Canvas 2D | Excalibur 0.32 | When you want a lightweight TypeScript 2D engine with scene management and built-in input. Still 0.x API instability. Viable alternative but adds abstraction overhead for minimal gain in a ~500-line game. |
| Pointer Events (native) | Hammer.js or custom touch library | The W3C Pointer Events Level 2 API (shipped in all modern browsers since 2020) unifies mouse, touch, and pen. Adding a library for input adds bundle size and abstraction cost for no benefit. The W3C Touch Events Community Group "strongly encourages adoption of Pointer Events" — touch events are a legacy API. |
| LocalStorage | Dexie (IndexedDB) or lokijs | When you need structured querying, large datasets, or complex schemas. For a single `highScore` number and optional settings, LocalStorage is the correct tool. Dexie adds ~12KB for no benefit at this scale. |
| Custom physics (projectile motion) | Matter.js / Planck.js / Cannon-es | When you need rigid body collisions, friction, restitution, and complex constraint solving. Archery trajectory is a single quadratic equation — integrating a physics engine for this would be cargo-culting. |
| Vite 8 | Webpack 5 | For new projects in 2026, there is no reason to choose Webpack. Vite is faster in every dimension (dev startup, HMR, build) and simpler to configure. |
| vite-plugin-pwa | Hand-written service worker | vite-plugin-pwa generates build-versioned Workbox 7 service workers with proper cache invalidation. Hand-writing a SW is error-prone and requires manual version management. The plugin handles SW lifecycle, precache manifest generation, and runtime caching with ~5 lines of config. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **React or any DOM framework** | Using React with Canvas creates a fundamental impedance mismatch. Game state (60fps updates) lives in refs/mutable objects, not React state. The dual-rendering pipeline (React reconciler + Canvas RAF) adds complexity with zero benefit. The game has ~3 DOM UI elements (menu, HUD, results) — vanilla DOM manipulation is simpler and faster. | Vanilla TypeScript + Canvas 2D + DOM overlays for UI |
| **Touch Events API** | Legacy API (W3C Touch Events Community Group Final Report, 2024) explicitly designated as legacy. Requires `{ passive: false }` which Chrome 136 broke for touch events (scroll prevention regression, May 2025). Pointer Events handles mouse+touch+pen uniformly. | Pointer Events API with `touch-action: none` CSS |
| **setInterval / setTimeout for game loop** | Not synced to display refresh, causes frame stutter, runs in background tabs (wastes battery on mobile). RAF auto-pauses when tab is hidden. | requestAnimationFrame with delta-time normalization |
| **CSS `scale()` for HiDPI** | CSS `scale()` transforms the canvas visually but doesn't change backing store resolution — results in blurry rendering on Retina displays. | `canvas.width = rect.width * dpr` + `ctx.scale(dpr, dpr)` |
| **Webpack / esbuild / Parcel** | Vite is the unequivocal standard for new TS projects in 2026. Webpack has 10x config overhead. esbuild lacks HMR. Parcel has smaller ecosystem. | Vite |
| **Howler.js / Web Audio libraries** | No sound in MVP (explicitly deferred in PROJECT.md Out of Scope). When sound is added, use raw Web Audio API (AudioContext + OscillatorNode for simple SFX) or a minimal library like zzfx (~2KB). | None (deferred) |

## Stack Patterns by Variant

**If you later add sound effects:**
- Use raw Web Audio API with `AudioContext` + `OscillatorNode` + `GainNode` for procedural SFX (arrow whoosh, hit thud, score chime). Procedural audio avoids audio file loading, caching, and format compatibility issues.
- Alternatively: zzfx (~2KB, procedural sound synthesis) if you want prebuilt instruments.

**If you later add a leaderboard (v2 scope):**
- Add `@supabase/supabase-js` for the backend, Supabase handles auth + database + REST API in one service.
- Keep LocalStorage as local cache, Sync to Supabase when online.

**If canvas rendering becomes a bottleneck (unlikely for this game):**
- Profile with Chrome DevTools Performance tab before optimizing.
- If needed: object pooling for arrow entities, skip `clearRect` and draw background only on dirty regions, or switch to off-screen canvas for static elements (target face, background).

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| TypeScript 6.0.x | Vite 8.x | Vite 8 uses esbuild for transpilation (no tsc). TS 6.0 uses own compiler for type-checking only. No compatibility issues. |
| vite-plugin-pwa 1.3.x | Vite 8.x | Latest plugin supports Vite 8. Specify `vite-plugin-pwa@latest` to get v1.3+. |
| Vite 8.0.x | Node 18+ | Vite 8 drops Node 16 support. Ensure dev environment uses Node 18 or 20 LTS. |

## Sources

- **npm registry** — verified versions: TypeScript 6.0.3, Vite 8.0.16, vite-plugin-pwa 1.3.0, Phaser 4.1.0, PixiJS 8.19.0, workbox-window 7.4.1
- **MDN Canvas optimization guide** (`/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas`) — HiDPI scaling pattern confirmed: `canvas.width = rect.width * dpr; canvas.height = rect.height * dpr; ctx.scale(dpr, dpr)` [HIGH confidence]
- **web.dev High DPI Canvas** (`/articles/canvas-hidipi`) — same DPR scaling pattern documented by Google as standard practice [HIGH confidence]
- **MDN Using Pointer Events** (`/en-US/docs/Web/API/Pointer_events/Using_Pointer_events`) — Pointer Events for unified input, `touch-action: none` CSS property [HIGH confidence]
- **W3C Touch Events Community Group** (2024 Final Report) — "strongly encourages adoption of Pointer Events", designates Touch Events as legacy [HIGH confidence]
- **insertcoin** (github.com/apratico/insertcoin, updated May 2026) — production reference: 27+ mobile-first Canvas 2D games in vanilla TS + Vite, zero game engines [HIGH confidence]
- **GamineAI "Game Development with TypeScript"** (gamineai.com, March 2026) — standard project structure for TS + Vite browser games: `src/core/` for engine utilities, `src/game/` for game logic, InputManager pattern [MEDIUM confidence — single source but aligns with all other findings]
- **MDN Touch events for mobile** (`/en-US/docs/Games/Techniques/Control_mechanisms/Mobile_touch`) — touch event handling patterns, preventDefault requirement, Chrome 136 regression [HIGH confidence]
- **Signature pad issue #825** (github.com/szimek/signature_pad, May 2025) — documented Chrome 136 breakage where passive touch event listeners no longer prevent scroll; fix: explicit `{ passive: false }` [HIGH confidence]
- **Vite + Vite-PWA guide** (2026) — PWA config pattern, Workbox 7 integration in vite-plugin-pwa [MEDIUM confidence — single source but aligns with plugin official docs]
- **LogRocket "Best JS game engines 2025"** (blog.logrocket.com, May 2025) — industry survey confirming Phaser, PixiJS, and vanilla Canvas positions [MEDIUM confidence — survey article, aligns with codebase evidence]
- **Phaser TypeScript + Vite guide** (generalistprogrammer.com, 2026) — confirms Vite as fastest build tool for game dev, sub-200ms reload [MEDIUM confidence — tutorial source, aligns with all other sources]

---
*Stack research for: Mobile-first HTML5 Canvas 2D arcade archery game*
*Researched: 2026-06-08*
