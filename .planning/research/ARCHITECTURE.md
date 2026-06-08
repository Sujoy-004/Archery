# Architecture Research

**Domain:** HTML5 Canvas 2D arcade game (archery bow & arrow)
**Researched:** 2026-06-08
**Confidence:** HIGH (patterns verified against multiple authoritative sources)

## Standard Architecture

### System Overview

The architecture follows the canonical **Process Input → Update → Render** game loop pattern, separated into four horizontal layers: Input, Game State, Simulation, and Presentation. A state machine governs which components are active per screen.

```
┌──────────────────────────────────────────────────────────────────┐
│                        SCREEN MANAGER                            │
│              (Stack-based state machine)                         │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ MenuState  │  │ PlayState    │  │ ResultsState │             │
│  │  - enter() │  │  - enter()   │  │  - enter()   │             │
│  │  - update()│  │  - update()  │  │  - update()  │             │
│  │  - render()│  │  - render()  │  │  - render()  │             │
│  │  - exit()  │  │  - exit()    │  │  - exit()    │             │
│  └────────────┘  └──────┬───────┘  └──────────────┘             │
│                          │ delegates to                           │
├──────────────────────────┴───────────────────────────────────────┤
│                       GAME LOOP                                   │
│  requestAnimationFrame + fixed timestep (60Hz) + accumulator      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  update(dt) {                                              │  │
│  │    input.poll();                                           │  │
│  │    stateManager.update(dt);  // delegates to active state  │  │
│  │  }                                                         │  │
│  │  render(alpha) {                                           │  │
│  │    stateManager.render(ctx, alpha);  // delegates to state │  │
│  │  }                                                         │  │
│  └─────────────────────────────────────────────────────────────┘  │
├───────────────────────────────────────────────────────────────────┤
│                     SIMULATION LAYER                               │
│  ┌──────────────┐  ┌────────────────┐  ┌───────────────────┐      │
│  │ Trajectory   │  │ Collision      │  │ EffectsSystem     │      │
│  │ System       │  │ System         │  │ (particles,       │      │
│  │ (arrow path, │  │ (hit detection │  │  trails, floating │      │
│  │  gravity)    │  │  → ring calc)  │  │  score text)      │      │
│  └──────┬───────┘  └───────┬────────┘  └────────┬──────────┘      │
│         │                  │                    │                 │
├─────────┴──────────────────┴────────────────────┴────────────────┤
│                       PRESENTATION LAYER                          │
│  ┌────────────┐  ┌─────────────────┐  ┌──────────────────┐       │
│  │ Canvas     │  │ DOM Overlay     │  │ Storage          │       │
│  │ Renderer   │  │ (menu, results) │  │ (high score via  │       │
│  │ (gameplay) │  │                 │  │  LocalStorage)   │       │
│  └────────────┘  └─────────────────┘  └──────────────────┘       │
├───────────────────────────────────────────────────────────────────┤
│                         INPUT LAYER                                │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  InputHandler (touch + mouse → normalized pointer state)    │  │
│  │  - pointerDown, pointerMove, pointerUp events              │  │
│  │  - dead zone, drag distance, aim offset                    │  │
│  │  - no direct state mutation — records, doesn't act         │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| `StateManager` | Holds state stack, delegates `update()`/`render()` to active state. Manages transitions (push/pop/replace). | Stack-based pushdown automaton. Each state is an object with `enter()`, `exit()`, `pause()`, `resume()`, `update(dt)`, `render(ctx, alpha)`. |
| `GameLoop` | Drives frame timing. Runs fixed-step update + interpolated render. | `requestAnimationFrame` with accumulator. Schedule rAF at top of loop. Clamp dt to 1000ms max. |
| `InputHandler` | Normalises touch/mouse to `{ x, y, isDown }`. Records pull distance, aim angle. Prevents browser defaults. | Unified touch/mouse event handlers with `getEventPos()`. Sets state object, never calls game logic directly. |
| `TrajectorySystem` | Computes arrow path. Applies gravity each tick. Optionally precomputes full path on release. | `update(dt)` adds gravity to `vy`, integrates position. Arcade gravity (lower than real). |
| `CollisionSystem` | Detects arrow vs target intersection. Maps impact point to scoring ring. | Screen-space check at target plane. Distance from center → score value (10/7/5/3). Emits score event. |
| `Renderer` | Draws all canvas visuals in correct order. Manages draw layers. | 7-layer draw pipeline. Snaps coordinates to integers. Batches state changes. Uses `{ alpha: false }`. |
| `EffectsSystem` | Runs transient visuals: arrow trail, hit impact, floating score text, arrow stuck in target. | Particle-like objects with lifetime. Pooled allocation. Rendered as overlay on gameplay canvas. |
| `Storage` | Reads/writes high score via LocalStorage. Schema versioning. Error handling. | `try/catch` wrapped. `{ version, payload }` envelope. `isAvailable()` check at boot. |
| `UIManager` | Manages DOM-based screens (menu, results). Not involved during gameplay. | DOM overlay divs shown/hidden based on state. Minimised DOM work during active play. |

### State Machine Detail

The game uses a **stack-based state machine** with three top-level states:

```
          ┌──────────┐
          │  Boot    │ (setup canvas, assets, storage check)
          └────┬─────┘
               ↓
          ┌──────────┐
          │  Menu    │ (title + "Play" button)
          └────┬─────┘
               ↓ (tap Play)
          ┌──────────┐
          │  Play    │ ←── The only state with active gameplay
          │          │     Contains sub-state machine for arrow:
          │  ┌─────────────┐
          │  │ idle        │  (waiting for pull)
          │  │ aiming      │  (drag in progress)
          │  │ flying      │  (arrow in flight)
          │  │ scoring     │  (hit feedback shown)
          │  └─────────────┘
          └────┬─────┘
               ↓ (3 arrows used)
          ┌──────────┐
          │ Results  │ (score + high score + restart/menu)
          └────┬─────┘
               ↓ (tap Menu or Restart)
          ┌──────────┐       ┌──────────┐
          │  Menu    │  or   │  Play    │
          └──────────┘       └──────────┘
```

**PlayState sub-states** (simpler — enum tracked within PlayState, not a full stack):

| Sub-state | Entry | Exit Condition |
|-----------|-------|----------------|
| `idle` | Round starts, arrow ready. | Pointer down on canvas → `aiming` |
| `aiming` | Bow pull, aim line visible. | Pointer up → `flying`; pointer cancel → `idle` |
| `flying` | Arrow follows trajectory. | Impact or off-screen → `scoring` |
| `scoring` | Show hit feedback, score pop. | Animation complete (0.8s) → `idle` or results |

## Recommended Project Structure

```
src/
├── main.ts                 # Bootstrap: init canvas, input, states, start loop
├── GameLoop.ts             # rAF + fixed timestep accumulator
├── StateManager.ts         # Stack-based state machine
├── states/
│   ├── MenuState.ts        # Title + play button (DOM overlay)
│   ├── PlayState.ts        # Gameplay: sub-state machine + orchestrator
│   └── ResultsState.ts     # Score display + restart/menu (DOM overlay)
├── input/
│   └── InputHandler.ts     # Unified touch/mouse → normalized state
├── physics/
│   ├── TrajectorySystem.ts # Arrow motion with gravity
│   └── CollisionSystem.ts  # Target hit detection + ring scoring
├── render/
│   ├── Renderer.ts         # Draw pipeline orchestrator
│   ├── layers/
│   │   ├── BackgroundLayer.ts
│   │   ├── TargetLayer.ts
│   │   ├── ArrowLayer.ts   # In-flight arrow only
│   │   ├── BowLayer.ts     # Bow + string visualization
│   │   └── EffectsLayer.ts # Particles, trails, floating text
│   └── effects/
│       ├── ArrowTrail.ts
│       ├── HitImpact.ts
│       └── FloatingScore.ts
├── entities/
│   ├── Bow.ts              # Bow position, draw state, string deformation
│   ├── Arrow.ts            # Arrow position, velocity, active/inactive
│   ├── Target.ts           # Ring positions, colors, scoring lookup
│   └── StuckArrow.ts       # Arrow embedded in target after hit
├── ui/
│   ├── UIManager.ts        # DOM overlay show/hide
│   ├── MenuScreen.ts       # Menu DOM construction
│   └── ResultsScreen.ts    # Results DOM construction
├── storage/
│   └── Storage.ts          # LocalStorage read/write with schema versioning
├── constants.ts            # All magic numbers: dimensions, speeds, gravity, ring sizes
└── types.ts                # Shared TypeScript types and interfaces
```

### Structure Rationale

- **`states/`**: Each screen is a self-contained state object. Adding new screens (settings, tutorial) means adding a state file, not touching existing code.
- **`physics/`**: Isolated from rendering. Can be unit-tested without a canvas. No DOM dependency.
- **`render/layers/`**: Each layer independently drawable. The draw order is enforced by the `Renderer` orchestrator, not by layer interdependencies. Layers are stateless — they receive state and draw it.
- **`entities/`**: Pure data objects. Arrow doesn't draw itself — `ArrowLayer` reads it and draws. Keeps data and presentation decoupled (this is the key separation: "entity data" vs "entity rendering").
- **`ui/`**: DOM-based, separate from canvas renderer. Only active during menu/results screens, never pollutes gameplay frames.
- **`input/`**: Single source of truth for pointer state. Every frame, `update()` reads `InputHandler` state and acts on it. No event handler mutates game state directly.

## Build Order & Dependencies

The build order is driven by dependency arrows: a component cannot be built until the components it depends on exist. This section informs the phase structure of the roadmap.

### Dependency Graph

```
Layer 1 (Foundation)    Layer 2 (Core)       Layer 3 (Gameplay)    Layer 4 (Screens)
────────────────────    ───────────────      ─────────────────     ─────────────────
constants.ts     ──────→ types.ts
                                    │
GameLoop.ts ←───────────────────────┤
                                    │
InputHandler.ts ←───────────────────┤
                                    │
Storage.ts (independent) ──────────→│
                                    │
StateManager.ts ←───────────────────┤
                                    ↓
                            PlayState.ts ──→ TrajectorySystem.ts
                                  │              │
                                  │              CollisionSystem.ts
                                  │
                                  ├─→ Bow.ts / Arrow.ts / Target.ts (entities)
                                  │
                                  ├─→ Renderer.ts
                                  │     ├── BackgroundLayer.ts
                                  │     ├── TargetLayer.ts
                                  │     ├── ArrowLayer.ts
                                  │     ├── BowLayer.ts
                                  │     └── EffectsLayer.ts
                                  │           ├── ArrowTrail.ts
                                  │           ├── HitImpact.ts
                                  │           └── FloatingScore.ts
                                  │
                                  └─→ EffectsSystem.ts
                                          │
                                          ↓
                                  MenuState.ts ──→ UIManager.ts → MenuScreen.ts
                                  ResultsState.ts ──→ UIManager.ts → ResultsScreen.ts
```

### Suggested Phase Order

| Phase | Builds | Depends On | Delivers |
|-------|--------|------------|----------|
| **Phase 1: Skeleton** | `constants.ts`, `types.ts`, `GameLoop.ts`, `StateManager.ts` | Nothing | App boots, shows a blank canvas at 60 FPS, states can be pushed |
| **Phase 2: Input + Bow** | `InputHandler.ts`, `entities/Bow.ts`, `InputHandler` | `GameLoop`, `types` | Player sees bow, can touch and drag, bow string deforms |
| **Phase 3: Target** | `entities/Target.ts`, `render/layers/TargetLayer.ts`, `render/layers/BackgroundLayer.ts` | `Renderer` pattern | Static target appears with 4 scoring rings |
| **Phase 4: Shoot** | `TrajectorySystem.ts`, `Arrow.ts`, `ArrowLayer.ts`, `BowLayer.ts` | Trajectory on `types`, Arrow on Bow pos | Player can pull and release → arrow flies with arc |
| **Phase 5: Hit Detection** | `CollisionSystem.ts` (scoring logic), `PlayState.ts` sub-state machine | Trajectory, Target, Arrow | Arrow hits target → score calculated |
| **Phase 6: Feedback** | `EffectsSystem.ts` (trail, impact, floating score), `StuckArrow.ts` | Collision system | Arrow trail, hit particles, floating score text, arrow sticks |
| **Phase 7: Screens** | `MenuState.ts`, `ResultsState.ts`, `UIManager.ts`, `Storage.ts` | StateManager, Storage | Menu → gameplay → results flow works end-to-end |
| **Phase 8: Polish** | HUD refinement, touch dead zone tuning, DPI handling, edge cases | Everything above | Ship-ready |

### Dependency Rules

- `GameLoop` is a standalone utility — no game-specific knowledge. Can be built and tested first.
- `InputHandler` is standalone — no game-specific knowledge. Can be built first.
- `Storage` is standalone — no game-specific knowledge. Can be built first.
- `TrajectorySystem` operates on `ArrowState` data — no dependency on rendering.
- `CollisionSystem` operates on position data — no dependency on rendering.
- `Renderer` and its layers depend on entity types but not on input or physics logic.
- `PlayState` is the integration point — depends on everything. Cannot be built until at least trajectory, target, and renderer exist.
- `MenuState` and `ResultsState` are UI-only — depend on `StateManager` and `UIManager` only.
- `Storage` must exist before `ResultsState` can save/load high score.

## Architectural Patterns

### Pattern 1: Fixed Timestep Game Loop with Interpolation

**What:** A game loop that decouples update frequency from render frequency. Update runs at a fixed rate (60 Hz) regardless of display refresh rate. Render runs whenever the browser paints, interpolating between updates for smooth visuals.

**When to use:** Any game with physics or deterministic simulation. For this archery game: trajectory simulation must behave identically at 30, 60, 120 FPS.

**Trade-offs:**
- + Deterministic physics across all devices
- + Frame-rate-independent gameplay
- + Stable collision detection (no missed frames from large dt)
- − Slightly more complex than variable timestep
- − Overkill if there's no physics (but we have projectile motion)

**Example:**
```typescript
// GameLoop.ts — Core loop with fixed timestep
const TIMESTEP_MS = 1000 / 60;  // 16.67ms
const MAX_FRAME_MS = 1000;      // clamp to avoid spiral of death

let lastTime = 0;
let accumulator = 0;
let frameId = 0;

function mainLoop(timestamp: number): void {
    frameId = requestAnimationFrame(mainLoop);  // Schedule early — critical

    const frameTime = Math.min(timestamp - lastTime, MAX_FRAME_MS);
    lastTime = timestamp;
    accumulator += frameTime;

    // Fixed-step updates
    let steps = 0;
    while (accumulator >= TIMESTEP_MS) {
        update(TIMESTEP_MS / 1000);  // dt in seconds
        accumulator -= TIMESTEP_MS;
        if (++steps >= 240) {        // panic guard
            accumulator = 0;
            break;
        }
    }

    // Interpolation factor for smooth rendering
    const alpha = accumulator / TIMESTEP_MS;
    render(alpha);
}

function start(): void {
    // Bootstrap: use a throwaway frame to initialise timing
    frameId = requestAnimationFrame((ts) => {
        lastTime = ts;
        frameId = requestAnimationFrame(mainLoop);
    });
}

function stop(): void {
    cancelAnimationFrame(frameId);
}
```

### Pattern 2: State Stack (Pushdown Automaton)

**What:** A stack of game states where only the topmost state receives `update()`/`render()` calls. States can push (overlay a pause menu on gameplay) or pop (resume gameplay). This project doesn't need pause, but the stack pattern gives clean screen transitions.

**When to use:** Any game with distinct screens or modes (menu → game → results). The stack is overkill for linear flow but future-proofs for overlays (pause, settings modal).

**Trade-offs:**
- + Clean separation of screen logic
- + Each state is independently testable
- + Stack allows overlays without coupling (e.g., pause over gameplay)
- − More files than a switch-statement approach
- − For a 3-state linear game, a simple enum + switch would also work

**Example:**
```typescript
// StateManager.ts — supports three screens (no pause in MVP)
interface GameState {
    enter?(): void;
    exit?(): void;
    update(dt: number): void;
    render(ctx: CanvasRenderingContext2D, alpha: number): void;
    handleInput?(input: InputState): void;
}

class StateManager {
    private stack: GameState[] = [];

    get current(): GameState | null {
        return this.stack[this.stack.length - 1] ?? null;
    }

    push(state: GameState): void {
        this.current?.pause?.();
        this.stack.push(state);
        state.enter?.();
    }

    pop(): GameState | undefined {
        const old = this.stack.pop();
        old?.exit?.();
        this.current?.resume?.();
        return old;
    }

    replace(state: GameState): void {
        this.stack.pop()?.exit?.();
        this.stack.push(state);
        state.enter?.();
    }

    update(dt: number): void {
        this.current?.update(dt);
    }

    render(ctx: CanvasRenderingContext2D, alpha: number): void {
        this.current?.render(ctx, alpha);
    }
}
```

### Pattern 3: Record-Transform-Render (Input Isolation)

**What:** A strict pipeline: (1) Input events record raw state into a buffer, (2) the `update()` phase transforms game state based on buffered input, (3) the `render()` phase only reads—never writes—game state.

**When to use:** Always. This is the fundamental discipline of game architecture. Violating it causes race conditions, missed inputs, and frame-dependent behaviour.

**Trade-offs:**
- + No race conditions between event handlers and game loop
- + Deterministic per-frame behaviour
- + Simple to debug: state only changes in `update()`
- − Slightly more indirection (event → buffer → action)

**Example:**
```typescript
// InputHandler.ts — records, never mutates
interface InputState {
    pointerX: number;
    pointerY: number;
    isDown: boolean;
    justPressed: boolean;
    justReleased: boolean;
}

class InputHandler {
    private state: InputState = {
        pointerX: 0, pointerY: 0,
        isDown: false, justPressed: false, justReleased: false,
    };

    private _justPressed = false;
    private _justReleased = false;

    init(canvas: HTMLCanvasElement): void {
        const getPos = (e: MouseEvent | TouchEvent) => {
            const rect = canvas.getBoundingClientRect();
            const clientX = 'touches' in e ? e.touches[0].clientX : e.clientX;
            const clientY = 'touches' in e ? e.touches[0].clientY : e.clientY;
            return { x: clientX - rect.left, y: clientY - rect.top };
        };

        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault();
            const pos = getPos(e);
            this.state.pointerX = pos.x;
            this.state.pointerY = pos.y;
            this.state.isDown = true;
            this._justPressed = true;
        }, { passive: false });

        canvas.addEventListener('touchmove', (e) => {
            e.preventDefault();
            const pos = getPos(e);
            this.state.pointerX = pos.x;
            this.state.pointerY = pos.y;
        }, { passive: false });

        canvas.addEventListener('touchend', (e) => {
            e.preventDefault();
            this.state.isDown = false;
            this._justReleased = true;
        }, { passive: false });

        // Mirror for mouse (desktop)
        canvas.addEventListener('mousedown', (e) => { /* same pattern */ });
        canvas.addEventListener('mousemove', (e) => { /* same pattern */ });
        canvas.addEventListener('mouseup', () => { /* same pattern */ });
    }

    /** Called once per frame at the start of update(). Flush transient flags. */
    poll(): InputState {
        this.state.justPressed = this._justPressed;
        this.state.justReleased = this._justReleased;
        this._justPressed = false;
        this._justReleased = false;
        return this.state;
    }
}

// In PlayState.update():
update(dt: number): void {
    const input = this.input.poll();

    switch (this.subState) {
        case 'idle':
            if (input.justPressed) this.startAim(input);
            break;
        case 'aiming':
            if (input.isDown) this.updateAim(input);
            else this.releaseArrow(input);
            break;
        case 'flying':
            this.trajectory.update(dt);
            if (this.collision.check()) this.showScoring();
            break;
        case 'scoring':
            // wait for animation timer
            break;
    }
}
```

### Pattern 4: Layered Render Pipeline

**What:** Render scene in strict back-to-front order. Each layer is drawn fully before the next begins. Layers correspond to visual depth planes, not entity types.

**When to use:** Every Canvas 2D game. There is no z-index in Canvas — draw order is the only depth control.

**Trade-offs:**
- + Simple, predictable, easy to debug
- + Layers can be toggled on/off (e.g., hide effects layer during low-FPS mode)
- + No z-ordering calculations needed
- − All layers are fully redrawn each frame (acceptable for this complexity level)
- − For 1000+ entities, would need spatial partitioning, but not applicable here

**Draw order for this game:**
```typescript
// Renderer.ts — strict 7-layer pipeline
render(ctx: CanvasRenderingContext2D, alpha: number): void {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    ctx.fillStyle = '#87CEEB';  // Hardcoded gradient or pre-rendered
    ctx.fillRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    // Layer 1: Background (sky, distant hills)
    this.backgroundLayer.draw(ctx);

    // Layer 2: Ground / environment
    this.groundLayer.draw(ctx);

    // Layer 3: Target (always visible in background plane)
    this.targetLayer.draw(ctx);

    // Layer 4: Arrow in flight (behind bow to show depth)
    if (this.arrow.active) {
        this.arrowLayer.draw(ctx, this.arrow, alpha);
    }

    // Layer 5: Bow + string (foreground, overrides arrow when pulled back)
    this.bowLayer.draw(ctx, this.bow, alpha);

    // Layer 6: HUD — score, arrows remaining
    // NOTE: Per TRD, HUD uses DOM overlay, not canvas
    // If using canvas HUD: this.hudLayer.draw(ctx, this.score);

    // Layer 7: Effects (trail, hit impact, floating score)
    this.effectsLayer.draw(ctx, alpha);
}
```

### Pattern 5: Trajectory with Gravity

**What:** A projectile motion system that applies gravity to the arrow's velocity each update. Arrow starts at bow position with velocity derived from pull distance and aim angle.

**When to use:** Any game with ballistic projectile physics. For archery, we want arcade-friendly gravity (not realistic — realistic archery arrows fly nearly straight at game distances).

**Trade-offs:**
- + Simple, well-understood physics
- + Easy to tune (adjust gravity constant until it "feels right")
- + Pre-computation possible (cache the full arc on release for deterministic playback)
- − Continuous simulation can desync on very low FPS (mitigated by fixed timestep)

**Example:**
```typescript
// TrajectorySystem.ts
const GRAVITY = 800;  // pixels/s² — tuned for arcade feel, not real physics

interface ArrowState {
    x: number;
    y: number;
    vx: number;
    vy: number;
    prevX: number;   // for interpolation
    prevY: number;   // for interpolation
    active: boolean;
}

class TrajectorySystem {
    update(arrow: ArrowState, dt: number): void {
        if (!arrow.active) return;

        // Store previous position for interpolation
        arrow.prevX = arrow.x;
        arrow.prevY = arrow.y;

        // Integrate
        arrow.vy += GRAVITY * dt;
        arrow.x += arrow.vx * dt;
        arrow.y += arrow.vy * dt;

        // Deactivate if off-screen
        if (arrow.y > CANVAS_HEIGHT || arrow.x > CANVAS_WIDTH || arrow.x < 0) {
            arrow.active = false;
        }
    }

    /** Launch the arrow from bow position with velocity from pull + aim */
    launch(arrow: ArrowState, pullDistance: number, aimAngle: number): void {
        const power = pullDistance * POWER_SCALE;  // linear or quadratic mapping
        arrow.x = BOW_X;
        arrow.y = BOW_Y;
        arrow.prevX = BOW_X;
        arrow.prevY = BOW_Y;
        arrow.vx = Math.cos(aimAngle) * power;
        arrow.vy = Math.sin(aimAngle) * power;
        arrow.active = true;
    }
}
```

## Data Flow

### Request Flow — "Player Takes a Shot"

```
[Player touches canvas]
    ↓
InputHandler.touchstart → records { pointerX, pointerY, isDown: true }
    ↓
[Next frame: PlayState.update()]
    ↓
InputHandler.poll() → { justPressed: true, pointerX, pointerY }
    ↓
PlayState transitions to 'aiming' sub-state
Sets bow pull start point
    ↓
[Touch move frames repeat]
InputHandler.poll() → updates aim angle + pull distance
Bow string deforms, arrow moves back with finger
    ↓
[Player lifts finger]
InputHandler.touchend → { justReleased: true }
    ↓
PlayState.releaseArrow()
    ↓
TrajectorySystem.launch(arrow, pullDistance, aimAngle)
→ Sets arrow.vx, arrow.vy from power + angle
→ PlayState transitions to 'flying'
    ↓
[flying frames repeat]
TrajectorySystem.update(arrow, dt) → gravity + integration
Renderer renders arrow at interpolated position
    ↓
CollisionSystem.check(arrow, target)
→ Distance from target center → score value (10/7/5/3)
→ If miss: arrow flies off-screen, deactivates
    ↓
[Hit or miss detected]
PlayState transitions to 'scoring'
EffectsSystem triggers: hit impact + floating score
Score total updated
    ↓
[0.8s delay, animation completes]
If arrows remaining > 0 → back to 'idle'
If all 3 arrows used → StateManager.replace(ResultsState)
```

### State Management

```
[StateManager]
    ↓ (push/replace)
┌──────────────────────────────────────────────────┐
│              ACTIVE STATE                         │
│  ┌──────────────────────────────────────────────┐ │
│  │ update(dt):                                 │ │
│  │   1. InputHandler.poll() → read input       │ │
│  │   2. Evaluate sub-state transitions         │ │
│  │   3. TrajectorySystem.update() if flying    │ │
│  │   4. CollisionSystem.check() if flying      │ │
│  │   5. EffectsSystem.update()                 │ │
│  └──────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────┐ │
│  │ render(alpha):                               │ │
│  │   1. Renderer.background()                   │ │
│  │   2. Renderer.ground()                       │ │
│  │   3. Renderer.target()                       │ │
│  │   4. Renderer.arrow()  (interpolated)        │ │
│  │   5. Renderer.bow()                          │ │
│  │   6. Renderer.effects()                      │ │
│  │   (DOM overlays: UI screen if menu/results)  │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### Key Data Flows

1. **Shot flow:** `InputHandler.poll()` → `PlayState` computes aim → `TrajectorySystem.launch()` sets arrow velocity → per-frame `TrajectorySystem.update()` advances position → `CollisionSystem.check()` tests hit → `PlayState` updates score → `StateManager` triggers results.

2. **Storage flow:** `ResultsState.enter()` → compare score vs `Storage.load('highScore')` → if higher, `Storage.save('highScore', score)` → display both in DOM.

3. **UI flow:** `StateManager` replaces state → new state's `enter()` → `UIManager` shows DOM overlay for that screen → previous state's DOM hidden.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| This game (simple arcade) | Architecture described above is sufficient. Single Canvas 2D, single-threaded, no network. |
| Adding multiplayer (v2) | Would need state serialization, network layer, prediction. Entirely separate concern — architecture above is unchanged; a network component wraps game state. |
| Adding leaderboards (v2) | Add API client module communicating with a REST backend. Storage layer extended with `LeaderboardAPI.ts`. No change to game loop. |
| Adding sound effects (future) | Add `AudioManager` with preloaded buffers. Triggered from PlayState on collision events. No architectural impact — decoupled via event dispatch. |

### Scaling Priorities

1. **First bottleneck:** Frame drops during heavy effects (particle trail + hit impact). Mitigation: pool particles, limit max particles, skip effects layer if FPS drops below 30.
2. **Second bottleneck:** Canvas resolution on very high-DPI devices (4k screens). Mitigation: cap internal resolution to 1080p, scale CSS if needed.

## Anti-Patterns

### Anti-Pattern 1: Mutating Game State in Event Handlers

**What people do:** `canvas.addEventListener('touchend', () => { arrow.vx = ...; arrow.vy = ...; })`

**Why it's wrong:** Race condition with the update loop. The arrow launch is not synchronised with the physics step. On fast devices the arrow may launch twice; on slow devices the arrow may not launch at all this frame. Also makes the game non-deterministic — same input, different outcome.

**Do this instead:** Event handlers only record input. All game state changes happen in `update()`, which reads input state once per frame.

### Anti-Pattern 2: Pre-computing Full Arrow Path in One Go

**What people do:** On release, run a loop that calculates the entire trajectory into an array of points, then animate through them.

**Why it's wrong:** (1) Cannot respond to dynamic changes (future wind, moving target in v2). (2) Creates a large array that's garbage-collected. (3) The animation fails if it desyncs from frame timing. (4) More complex than needed.

**Do this instead:** Simulate continuously. Arrow gets `vx`, `vy` on launch. Each `update()` applies gravity and integrates. It's 2 lines of math per frame. Simpler, flexible, and uses no allocations.

### Anti-Pattern 3: Using `save()`/`restore()` for Every Shape

**What people do:** Wrapping every `draw()` call in `ctx.save(); ...draw... ctx.restore();` "just in case."

**Why it's wrong:** `save()`/`restore()` are surprisingly expensive — they snapshot and restore the entire Canvas state stack (transforms, styles, clipping, etc.). Doing it per-entity multiplies the cost.

**Do this instead:** Only use `save()`/`restore()` when you apply transforms (translate, rotate, scale) or clip paths that must be undone. For style changes (`fillStyle`, `strokeStyle`), set the property directly — the next draw call sets its own style anyway.

### Anti-Pattern 4: Hiding Everything in One "Canvas Engine" Class

**What people do:** Creating a monolithic `GameEngine` or `CanvasEngine` class that owns the loop, renders everything, handles input, runs physics, and manages state.

**Why it's wrong:** Tightly couples everything. Can't test physics without rendering. Can't change rendering without risking physics. Single-file becomes 2000+ lines. Any bug cascades.

**Do this instead:** Small, single-purpose modules with explicit interfaces. The `GameLoop` orchestrates when `update()` and `render()` are called but doesn't care what they do. `PlayState` orchestrates game systems but doesn't manage the canvas. `Renderer` draws but doesn't know about input or physics.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| None for MVP | Entirely client-side | No network, no backend, no analytics |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| `InputHandler` → `StateManager` | Input state object (read per frame) | One-way: state manager reads input; input never calls state manager |
| `StateManager` → `Renderer` | Active state's `render(ctx, alpha)` call | Rendering never writes to state |
| `PlayState` → `TrajectorySystem` | Method call: `.launch()`, `.update()` | Trajectory system is stateless — it operates on the arrow data passed in |
| `PlayState` → `CollisionSystem` | Method call: `.check(arrow, target)` → score or miss | Collision system is stateless — returns result, stores nothing |
| `PlayState` → `EffectsSystem` | Event-like: `.trigger('hit', position)`, `.trigger('score', value)` | Effects system manages its own particle pool, lifetime |
| `ResultsState` → `Storage` | Method calls: `.load('highScore')`, `.save('highScore', n)` | Storage is synchronous (LocalStorage). Wrapped in try/catch. |
| `UIManager` ↔ `DOM elements` | DOM manipulation (class toggles, text content) | Only active during menu/results states. Never during gameplay frames. |

## Sources

- [MDN: Anatomy of a Video Game](https://developer.mozilla.org/en-US/docs/Games/Anatomy) — HIGH confidence. Core loop patterns and browser timing. (2026-02-27)
- [Claude Code Forge: HTML5 Canvas Game Development Best Practices Reference](https://github.com/Clazman55/claude-code-forge/blob/main/skills/html5-canvas-game-development.md) — HIGH confidence. Comprehensive patterns for game loop, state management, input, rendering, storage. (2025-2026)
- [Game Programming Patterns: State](https://gameprogrammingpatterns.com/state.html) — HIGH confidence. Canonical reference for stack-based state machines.
- [Spicy Yoghurt: Create a Proper Game Loop](https://spicyyoghurt.com/tutorials/html5-javascript-game-development/create-a-proper-game-loop-with-requestanimationframe) — MEDIUM confidence. Tutorial coverage matches MDN.
- [Tidbytez: Building a Scalable 2D Game Scene Architecture](https://tidbytez.com/2025/10/01/building-a-scalable-2d-game-scene-architecture-from-back-to-front/) — MEDIUM confidence. Layer organization patterns for 2D games. (2025-10-01)
- [Korat Ozturan: requestAnimationFrame Done Right](https://korato.net/b/requestanimationframe-game-loop-delta-time.html) — MEDIUM confidence. Delta time clamping and spiral-of-death prevention. (2026-02-07)
- [O'Reilly: The Game State Machine — HTML5 Canvas 2nd Edition](https://learning.oreilly.com/library/view/html5-canvas-2nd/9781449335847/ch08s09s01.html) — MEDIUM confidence. State machine patterns for HTML5 Canvas games.

---
*Architecture research for: Archery Game (HTML5 Canvas 2D)*
*Researched: 2026-06-08*
