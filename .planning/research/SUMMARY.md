# Project Research Summary

**Project:** Archery Game
**Domain:** Mobile-first HTML5 Canvas 2D arcade archery game
**Researched:** 2026-06-08
**Confidence:** HIGH

## Executive Summary

This is a **mobile-first, single-player archery arcade game** — a casual browser game where players drag-to-pull a bow and release to shoot arrows at a stationary target. The genre is well-established with 15+ competing titles (Archery World Tour, Archery Master, Stickman Archer, etc.), and the expert consensus is unanimous: build it with **vanilla TypeScript + Vite + Canvas 2D** and zero game engines. The entire game logic fits in ~500-800 lines, making external dependencies like Phaser (450KB+) or PixiJS (800KB+) unjustified bloat. The architectural pattern is the canonical **fixed-timestep game loop + stack-based state machine + Record→Transform→Render input isolation** — the same proven pattern used by all browser games since the Canvas API standardized.

**The recommended approach:** Start with a TypeScript + Vite + vanilla Canvas 2D stack. Build in strict dependency order: skeleton → input + bow → target → shooting physics → hit detection → visual feedback → screens → polish. The "drag-to-release" shooting mechanic is the root dependency — everything else branches from it. Ship with the 10 table-stakes features (drag-to-shoot, ring target, shot feedback, 3-arrow round, results screen, high score, menu, arrow count display, touch scroll prevention, instant restart), then add polish features (wind physics, power meter, sound effects, combo multiplier) after the core loop is validated with real users.

**Key risks and mitigations:** (1) **GC stutter** from per-frame object allocation — solved by zero-allocation patterns (object pooling, `out` parameter pattern, `for` loops over `.filter()/.map()`) established from day one. (2) **Touch scroll interference** on mobile — solved by triple-layer prevention (CSS `touch-action: none`, viewport meta `user-scalable=no`, JS `{ passive: false }` on canvas listeners). (3) **Canvas blur on Retina/HiDPI displays** — solved by scaling canvas backing store by `window.devicePixelRatio`. (4) **Physics explosion on tab return** — solved by delta clamping (`Math.min(delta, 0.25)`) and `visibilitychange` handling. (5) **LocalStorage failure in Safari private mode** — solved by try-catch wrapping with in-memory fallback.

**⚠️ Research conflict identified:** PROJECT.md lists wind mechanics as "Out of Scope — adds complexity without proportional fun," while FEATURES.md research identifies wind as the "#1 most impactful depth addition" (P2 priority, recommended after core validation). The roadmap should treat wind as an **optional P2 that the orchestrator/solo developer can decide on after core loop validation** — not required for MVP, but strongly recommended by competitive analysis if the core loop feels shallow.

## Key Findings

### Recommended Stack

The stack research (STACK.md) recommends an **ultra-lean, zero-unnecessary-dependency** approach. The game is simple enough (~500-800 lines) that every external dependency is a liability, not an asset.

**Core technologies:**
- **TypeScript 6.0.x**: Static typing for game logic, state machines, and rendering — eliminates null refs and impossible states. The 2025/2026 standard for browser game projects.
- **Vite 8.0.x**: Industry-standard build tool — sub-300ms HMR, native ESM dev server, zero-config for vanilla TS + Canvas projects. Replaces Webpack entirely.
- **HTML5 Canvas 2D (Native API)**: Correct rendering choice for ~10 draw calls per frame. Avoids ~800KB+ bundle cost of WebGL engines. Universal mobile support.
- **requestAnimationFrame (Native API)**: Browser-synced vsync, automatic pause when tab hidden, frame-accurate timing. No library needed — ~20 lines of code.
- **vite-plugin-pwa 1.3.x** (dev dependency): Generates Workbox 7 service worker for offline support. Required by AUTH-09.

**What NOT to use:** React (impedance mismatch with Canvas), Touch Events API (legacy — use Pointer Events), setInterval for game loop (RAF is correct), CSS `scale()` for HiDPI (use `canvas.width * dpr`), any physics library (projectile motion is one quadratic equation), Howler.js (no sound in MVP), any state management library.

### Expected Features

The features research (FEATURES.md) analyzed 15+ competing archery games and identified a clear hierarchy:

**Must have (table stakes) — 10 features, all P1, product is broken without any:**
- Drag-to-pull-and-release shooting — THE universal control scheme, any alternative feels wrong
- Ring-based target with 4 rings (10/7/5/3) — standard scoring, well-calibrated per PROJECT.md
- Immediate shot feedback — arrow trail + impact embed + floating score text (all three required)
- 3-arrow round structure — short session constraint (<2 min per PRD)
- Results screen with final score, high score, restart, menu
- High score persistence via LocalStorage
- Simple main menu (title + Play button + high score display)
- Arrow count remaining display
- Mobile touch controls with no scroll/zoom
- Instant restart (<1 second from results to new round)

**Should have (competitive differentiation) — P2, add after core validation:**
- **Wind indicator + physics** — transforms shooting from "aim at center" into a reading+compensation skill puzzle. **⚠️ CONFLICT: PROJECT.md marks this as Out of Scope. See Executive Summary.**
- Power/draw strength meter — obvious UX improvement, players currently guess pull strength
- Sound effects — highest polish-per-effort ratio (bow twang, hit thud, score chime)
- Combo/streak multiplier — rewards consistency, adds tension

**Defer (v3+):**
- Multiple game modes (Balloon Challenge, Moving Targets, Time Attack) — significant scope
- Progressive levels (50 levels with 3-star rating) — requires level editor
- Trajectory prediction line — add if players struggle with aim
- Moving targets — introduce as harder levels
- Haptic feedback, customization, leaderboards, character skins

**Anti-features (explicitly avoid):** Ads in MVP (destroys premium feel), multiplayer (contradicts offline-first), login/accounts (backend friction for casual game), pay-to-win upgrades (invalidates skill-based scoring), auto-aim (undermines core value proposition), complex simulation physics (PRD says "arcade not simulation"), forced tutorials (mechanic should be self-evident), timed rounds as default (contradicts precision/calm satisfaction), punishing miss feedback (discourages retries), economy/shop/currencies (scope creep).

### Architecture Approach

The architecture (ARCHITECTURE.md) follows the canonical **Process Input → Update → Render** game loop pattern separated into four horizontal layers: Input, Game State, Simulation, and Presentation. A stack-based state machine governs three top-level states (Menu → Play → Results) with PlayState containing a simple sub-state machine (idle → aiming → flying → scoring).

**Major components:**
1. **GameLoop** — Fixed-timestep (60Hz) `requestAnimationFrame` with accumulator. Decouples update frequency from render frequency. Delta clamped to 250ms max to prevent physics explosion on tab return.
2. **StateManager** — Stack-based pushdown automaton. States: `MenuState`, `PlayState`, `ResultsState`. Supports push/pop/replace for clean screen transitions and future overlays (pause, settings).
3. **InputHandler** — Unified touch/mouse via Pointer Events. Records normalized pointer state per frame — never mutates game state directly (Record→Transform→Render pattern).
4. **TrajectorySystem** — Applies arcade gravity (800 px/s²) to arrow velocity each tick. Launch velocity derived from pull distance × aim angle. Stateless — operates on ArrowState data passed in.
5. **CollisionSystem** — Screen-space distance check at target plane. Maps impact point to scoring ring (10/7/5/3). Stateless — returns result, stores nothing. Emits score event.
6. **Renderer** — Strict 7-layer draw pipeline (background → ground → target → arrow → bow → HUD → effects). Layers are stateless — receive state and draw it. Batches draw calls, snaps to integer coordinates.
7. **EffectsSystem** — Manages transient visuals (arrow trail, hit impact, floating score text, stuck arrows). Pool-allocated, lifetime-managed particles.
8. **Storage** — LocalStorage wrapper with try-catch, schema versioning, in-memory fallback for Safari private mode.
9. **UIManager** — DOM overlay management for menu/results screens. Never active during gameplay frames (minimizes layout thrash).

**Key pattern: Fixed-timestep game loop with interpolation.** Update runs at fixed 60Hz regardless of display refresh rate. Render runs at display refresh rate, interpolating between updates. This ensures deterministic physics on 30Hz, 60Hz, and 120Hz devices.

### Critical Pitfalls

The pitfalls research (PITFALLS.md) identified 15 major pitfalls with detailed prevention strategies. Top 5 by impact:

1. **Per-frame GC allocations (Critical — causes stutter)** — Game loop allocating new objects every frame triggers V8 GC pauses (10-50ms), dropping 1-3 frames. Prevention: object pooling, `out` parameter pattern, `for` loops over `.filter()/.map()`, reset arrays with `.length = 0`. Establish zero-allocation discipline before adding particle effects.

2. **Touch scroll during gameplay (Critical — unplayable without fix)** — Mobile drag gesture causes page scroll unless prevented at CSS, meta, and JS layers. Prevention: triple-layer approach — `touch-action: none` CSS, `user-scalable=no` viewport meta, `{ passive: false }` on canvas touch listeners.

3. **Canvas blurry on Retina/HiDPI (Critical — looks unpolished)** — Canvas internal resolution defaults to CSS size, not physical pixels. Prevention: scale backing store by `window.devicePixelRatio`, use `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` (not cumulative `scale()`).

4. **Delta spike on tab return (Critical — physics explosion)** — 30-second tab switch delivers 30,000ms delta → 1,800 "frames" processed instantly. Prevention: cap delta at 250ms, pause accumulator on `visibilitychange`, reset `lastTime` on resume.

5. **iOS viewport height changes (Critical — content cut off)** — Safari address bar collapse/expand changes `window.innerHeight`. `100vh` includes bar area, causing blank strips. Prevention: dynamic viewport height via `--vh` CSS variable, `viewport-fit=cover`, `env(safe-area-inset-*)` padding, `position: fixed` canvas sizing.

**Additional noteworthy pitfalls:** LocalStorage throws in Safari private mode (wrap in try-catch), cumulative `ctx.scale()` on resize (use `setTransform()`), missing `touchcancel` handler (bow stuck on iOS system interruption), `drawImage` scaling every frame (use offscreen canvas caching), no frame budget monitoring on low-end devices (implement adaptive quality scaling).

## Implications for Roadmap

Based on combined research, the suggested phase structure follows strict dependency ordering from the architecture's dependency graph. Each phase builds exactly what the next phase depends on, with no backtracking.

### Phase 1: Skeleton (Foundation)

**Rationale:** The game loop and state manager are prerequisites for every other component. Constants and types define the shared vocabulary.
**Delivers:** App boots and shows a blank canvas at 60 FPS. States can be pushed/replaced. No visual content.
**Builds:** `constants.ts`, `types.ts`, `GameLoop.ts`, `StateManager.ts`
**Uses:** Stack: TypeScript 6, Vite 8 (scaffolding), Canvas 2D (no rendering yet), `requestAnimationFrame` with fixed-timestep accumulator.
**Avoids:** Pitfall 4 (delta clamping — establish in GameLoop from day one), Pitfall 7 (update/render separation), Pitfall 12 (visibilitychange handler).
**No research needed:** Well-documented fixed-timestep pattern. Standard boilerplate.

### Phase 2: Input + Bow (Core Interaction)

**Rationale:** The drag-to-release shooting mechanic is the root dependency of the entire game. Everything branches from this. Getting input right on mobile is the hardest technical challenge.
**Delivers:** Player sees a bow on canvas, can touch and drag, bow string deforms visually. No shooting yet.
**Builds:** `InputHandler.ts`, `entities/Bow.ts`, `BowLayer.ts` (render layer for bow)
**Uses:** Pointer Events API, `canvas.getBoundingClientRect()` for coordinate conversion, `touch-action: none` CSS + `{ passive: false }` listeners.
**Addresses features:** Drag-to-pull shooting mechanic (partial — pull registered, no release yet)
**Avoids:** Pitfall 2 (touch scroll — triple-layer prevention baked in), Pitfall 10 (coordinate conversion), Pitfall 13 (touchcancel handler)
**Research flag:** **Needs `/gsd-plan-phase --research-phase`** — touch coordinate conversion and iOS `touchcancel` behavior should be verified against real device testing. The InputHandler is the most platform-sensitive component.

### Phase 3: Target (Static Gameplay Element)

**Rationale:** Target is needed before any hit detection or scoring. Independent of input — can be built in parallel with Phase 2.
**Delivers:** Static target with 4 scoring rings appears on canvas. Background layer renders behind it.
**Builds:** `entities/Target.ts`, `render/layers/TargetLayer.ts`, `render/layers/BackgroundLayer.ts`, `Renderer.ts` (orchestrator), `GroundLayer.ts`
**Uses:** Canvas 2D (circles, arcs, fills). Pre-render target rings to offscreen canvas for caching.
**Addresses features:** Ring-based target with scoring (10/7/5/3 per AUTH-04)
**Avoids:** Pitfall 9 (drawImage scaling — cache static elements), Pitfall 14 (imageSmoothingEnabled)
**No research needed:** Standard Canvas 2D rendering. Well-established patterns.

### Phase 4: Shoot (Arrow Physics + Launch)

**Rationale:** Arrow needs the bow position (Phase 2) and the trajectory system (independent). Player can now complete the full interaction: pull → release → arrow flies.
**Delivers:** Player can pull bow and release → arrow flies with arcade gravity trajectory. No hit detection yet — arrow passes through target or goes off screen.
**Builds:** `TrajectorySystem.ts`, `Arrow.ts`, `ArrowLayer.ts`
**Uses:** Simple Euler integration with arcade gravity (800 px/s²). Velocity derived from pull distance × aim angle.
**Addresses features:** Drag-to-pull-and-release shooting (complete), arrow trail (partial — arrow path visible)
**Avoids:** Pitfall 1 (GC allocations — use pre-allocated ArrowState objects, no `new` in hot path)
**No research needed:** Projectile motion physics is ~5 lines of math. Direct implementation.

### Phase 5: Hit Detection + Scoring (Core Gameplay Loop)

**Rationale:** Collision detection needs trajectory system (Phase 4) and target entity (Phase 3). This is the integration point where standalone systems come together.
**Delivers:** Arrow hits target → score calculated and displayed. PlayState sub-state machine orchestrates the idle→aiming→flying→scoring flow.
**Builds:** `CollisionSystem.ts`, `PlayState.ts` (full implementation), score tracking logic
**Uses:** Screen-space distance check from arrow position to target center. Maps to scoring ring.
**Addresses features:** Score updates immediately on hit (AUTH-05), round ends after 3 arrows (AUTH-06)
**Research flag:** **Needs `/gsd-plan-phase --research-phase`** — PlayState is the most complex component, integrating input, physics, collision, rendering, and state transitions. Sub-state machine timing (0.8s scoring delay) needs tuning.

### Phase 6: Visual Feedback + Effects (Polish Layer)

**Rationale:** Effects system depends on collision detection (Phase 5) to trigger. Trail depends on arrow flight (Phase 4). Must be built after the core loop is verified.
**Delivers:** Arrow trail during flight, hit impact particles, floating score text, arrow stuck in target. Game starts to feel polished.
**Builds:** `EffectsSystem.ts`, `ArrowTrail.ts`, `HitImpact.ts`, `FloatingScore.ts`, `StuckArrow.ts`, `EffectsLayer.ts`
**Uses:** Pool-allocated particle system (prewarmed pools to avoid GC), lifetime-managed effects.
**Addresses features:** Arrow trail (AUTH-12), hit impact (AUTH-12), floating score text (AUTH-12), immediate shot feedback (complete — trail + impact + score text)
**Avoids:** Pitfall 1 (GC stutter — critical: effects are the #1 allocation hotspot), Pitfall 15 (adaptive quality — reduce particles on low-end)
**Research flag:** **Needs `/gsd-plan-phase --research-phase`** — Particle system design and pooling strategy. This is the most GC-sensitive phase.

### Phase 7: Screens + Storage (Full Flow)

**Rationale:** Menu and results screens depend on StateManager (Phase 1). Storage can be built independently. Results need storage for high score. The game is not a complete product without these screens.
**Delivers:** Main menu → gameplay → results → restart or menu. High score persists across sessions. Full user flow.
**Builds:** `MenuState.ts`, `ResultsState.ts`, `UIManager.ts`, `MenuScreen.ts`, `ResultsScreen.ts`, `Storage.ts`
**Uses:** DOM overlays for menu/results (per ARCHITECTURE.md Canvas 2D constraint). LocalStorage with try-catch fallback.
**Addresses features:** Simple main menu (AUTH-01), results screen (AUTH-07), high score persistence (AUTH-08), arrow count display, instant restart
**Avoids:** Pitfall 6 (localStorage in private mode), Pitfall 5 (iOS viewport sizing for DOM overlays)
**No research needed:** Standard patterns. Storage wrapper is well-documented.

### Phase 8: Polish + Ship (Production Readiness)

**Rationale:** All features implemented. This phase hardens for production: HiDPI scaling, DPR capping, performance monitoring, edge cases, touch dead zone tuning.
**Delivers:** Ship-ready game. 60 FPS on target devices. Crisp on Retina. No jank. Handles orientation changes, background tabs, interruptions.
**Builds:** DPR-aware resize handler, adaptive quality system, HUD refinement, input dead zone tuning, viewport/safe-area handling
**Avoids:** Pitfall 3 (HiDPI blur), Pitfall 5 (iOS viewport), Pitfall 8 (context reset on resize), Pitfall 11 (cumulative scale), Pitfall 15 (adaptive quality)
**No research needed:** Well-documented Canvas 2D hardening patterns.

### Phase 9+: Post-Validation Polish (v1.x)

**Rationale:** These features should only be built after the core loop is validated with real users. They are enhancements, not necessities.
**Delivers:** Wind physics (⚠️ see conflict note), power/draw meter, sound effects (Web Audio API), combo/streak multiplier
**Uses:** Web Audio API (procedural SFX via OscillatorNode/GainNode), trajectory drift calculation for wind
**Research flag:** **Needs `/gsd-plan-phase --research-phase`** — Wind physics requires tuning (magnitude, frequency, visual indicator). Sound design requires audio assets or procedural synthesis patterns.

### Phase Ordering Rationale

- **Strict dependency ordering:** The architecture dependency graph (ARCHITECTURE.md lines 176-212) dictates that GameLoop, InputHandler, Constants, and Types must come first. Physics and rendering cannot be tested until these exist. PlayState cannot work until trajectory, target, and renderer exist.
- **Input before gameplay:** The drag-to-release mechanic is the root dependency (FEATURES.md line 104 — "If the shooting doesn't feel good, nothing else matters"). Phase 2 (Input + Bow) is explicitly prioritized over Phase 3 (Target) because getting touch input right is the hardest technical challenge.
- **Screens last:** Menu and results are thin DOM overlays built after the gameplay loop is complete. Building them earlier adds no value without the core game to bridge between.
- **Polish as its own phase:** HiDPI scaling, viewport handling, and adaptive quality span multiple components. Dedicated Phase 8 addresses all edge cases together, preventing "works on my machine" syndrome.
- **P1 → P2 separation:** The 10 table-stakes features (Phase 1-8) are the MVP. The 4 P2 features (Phase 9+) are explicitly gated on core-loop validation. This prevents scope creep while preserving a clear enhancement path.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 2 (Input + Bow):** Touch coordinate conversion and `touchcancel` behavior need real-device verification. Platform-specific quirks (iOS vs Android, Chrome vs Safari) cannot be fully predicted from research.
- **Phase 5 (Hit Detection + Scoring):** PlayState integration is the most complex component. Sub-state machine timing and state transition edge cases need careful planning.
- **Phase 6 (Visual Feedback + Effects):** Particle pooling strategy and allocation budget need design upfront. This is the most GC-sensitive phase.
- **Phase 9+ (Post-Validation):** Wind physics tuning (magnitude, frequency, indicator design) and sound design (procedural vs sampled) need dedicated research when approached.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Skeleton):** Fixed-timestep game loop is a well-documented pattern. ~25 lines of code.
- **Phase 3 (Target):** Canvas circle rendering with pre-rendered offscreen caching. Standard pattern.
- **Phase 4 (Shoot):** Projectile motion physics (Euler integration with gravity). ~10 lines of code.
- **Phase 7 (Screens + Storage):** DOM overlay pattern + LocalStorage wrapper. Standard web patterns.
- **Phase 8 (Polish):** DPR scaling, viewport sizing, and canvas hardening. Well-documented on MDN.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | **HIGH** | Verified against npm registry (TypeScript 6.0.3, Vite 8.0.16, vite-plugin-pwa 1.3.0), MDN Canvas optimization guide, web.dev High DPI Canvas, W3C Pointer Events spec, and production reference (insertcoin — 27+ games). Multiple authoritative sources agree on vanilla TS + Vite + Canvas 2D. |
| Features | **HIGH** | Analyzed 15+ competitor games across HTML5 portals and mobile stores. Open-source references (4 GitHub repos) confirm patterns. User sentiment from app store reviews validates anti-feature analysis. The feature hierarchy (table stakes vs differentiators vs anti-features) is well-supported by evidence. |
| Architecture | **HIGH** | Patterns verified against MDN Anatomy of Video Games, Game Programming Patterns (canonical), Claude Code Forge game dev reference, and multiple tutorial sources. Fixed-timestep, state machine, layered rendering are all standard patterns with decades of game industry practice. |
| Pitfalls | **HIGH** | Verified against real-device benchmarks (GC stutter studies), MDN documentation (DPR, viewport, storage), WebKit bug tracker (localStorage in private mode), Chrome DevRel (frame budget), and community post-mortems. Each pitfall has actionable prevention with code examples. |

**Overall confidence: HIGH**

### Gaps to Address

1. **Wind physics conflict (PROJECT.md vs FEATURES.md):** PROJECT.md explicitly lists wind as Out of Scope ("adds complexity without proportional fun"). FEATURES.md identifies it as the "#1 most impactful depth addition" and P2 priority. **Resolution:** The roadmap treats wind as an optional P2 enhancement gated on core-loop validation. The orchestrator/solo developer decides after Phase 8 whether to proceed. If proceeding, the trajectory system should be designed with wind in mind (even if not implemented initially) to avoid refactoring.

2. **Sound effects timing:** PROJECT.md says "Sound is reserved for a future update." FEATURES.md lists sound as P2 with "highest polish-per-effort ratio." **Resolution:** Deferred to Phase 9+ (post-validation). No impact on MVP phases. When added, use raw Web Audio API (OscillatorNode + GainNode) or zzfx (~2KB) for procedural SFX.

3. **Real-device testing for touch input:** InputHandler is the most platform-sensitive component. Touch behavior differs between iOS Safari, Chrome Android, and Samsung Internet. **Resolution:** Phase 2 planning should include a device-testing matrix (minimum: iPhone SE, iPhone 14 Pro, Pixel 6, Samsung A-series).

4. **Performance budget for low-end devices:** The adaptive quality system (Pitfall 15) is flagged but not a specific requirement in PROJECT.md. **Resolution:** Phase 8 should include frame-time monitoring and quality degradation (reduce particles, cap DPR, skip effects) as a hardening step.

5. **No competitor analysis of difficulty/retention curves:** Features research analyzed feature sets but not engagement metrics. The 3-arrow round structure and single difficulty level may not provide enough retention. **Resolution:** Treat this as a validation question — ship MVP, measure if players replay, add progressive difficulty (P3) only if retention is insufficient.

## Sources

### Primary (HIGH confidence)
- **npm registry** — Verified versions: TypeScript 6.0.3, Vite 8.0.16, vite-plugin-pwa 1.3.0
- **MDN Canvas optimization guide** — HiDPI scaling, drawImage caching, context state management
- **web.dev High DPI Canvas** — `devicePixelRatio` scaling pattern
- **MDN Pointer Events** — Unified touch/mouse input, `touch-action: none`
- **W3C Touch Events Community Group (2024)** — Pointer Events adoption recommendation, Touch Events as legacy
- **insertcoin** (github.com/apratico/insertcoin) — Production reference: 27+ mobile-first HTML5 Canvas 2D games with vanilla TS + Vite
- **MDN Anatomy of Video Games** — Core loop patterns and browser timing
- **Game Programming Patterns (Robert Nystrom)** — Canonical state machine patterns
- **WebKit Bug #157010** — localStorage quota 0 in Safari private mode
- **Chrome DevRel: Speed Rendering** — Frame budget breakdown, jank causes

### Secondary (MEDIUM confidence)
- **Claude Code Forge: HTML5 Canvas Game Development Best Practices Reference** — Comprehensive architecture patterns
- **GamineAI "Game Development with TypeScript" (2026)** — Project structure conventions
- **LogRocket "Best JS game engines 2025"** — Industry survey confirming vanilla Canvas, Phaser, PixiJS positions
- **Competitor analysis:** Archery World Tour, Archery Master, Stickman Archer, Archery Elite, Archery Blast, Tiny Archer (15+ titles)
- **DEV Community: Zero-allocation TypeScript game loops (2026)** — Real-device GC benchmarks
- **Hex Hour: How HTML5 Canvas Performance Actually Works (2026)** — GPU-accelerated Canvas pipeline analysis
- **Korat Ozturan: requestAnimationFrame Done Right (2026)** — Delta clamping and spiral-of-death prevention
- **Spicy Yoghurt: Create a Proper Game Loop** — Fixed-timestep tutorial
- **TrackJS: Failed to execute 'setItem' on 'Storage'** — Safari private mode workaround
- **Polypane: Using safe-area-inset** — iOS safe area patterns

### Tertiary (LOW confidence — single source, aligns with higher-confidence findings)
- **Vite + Vite-PWA guide (2026)** — PWA config pattern, aligns with plugin official docs
- **Tidbytez: Building a Scalable 2D Game Scene Architecture (2025)** — Layer organization patterns
- **O'Reilly: The Game State Machine — HTML5 Canvas 2nd Edition** — State machine patterns
- **Bugnet: Game Save Best Practices (2026)** — localStorage pitfalls, persistent storage API

---

*Research completed: 2026-06-08*
*Ready for roadmap: yes*
