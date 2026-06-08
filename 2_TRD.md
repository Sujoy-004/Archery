# Technical Requirements Document

## 1. Technology Stack
- TypeScript
- Vite
- HTML5 Canvas 2D
- requestAnimationFrame
- LocalStorage

## 2. System Architecture
Use a small, explicit architecture with clear boundaries.

### Recommended Modules
- `main.ts` — bootstrapping and orchestration
- `GameLoop` — fixed update and render loop
- `InputHandler` — pointer/touch interpretation
- `TrajectorySystem` — arrow motion and physics
- `CollisionSystem` — hit validation and scoring
- `Renderer` — scene drawing
- `UI` — screens and overlays
- `storage` — local high score persistence

## 3. Architectural Principles
- Keep rendering separate from physics.
- Keep input separate from game state mutation.
- Keep collision separate from visuals.
- Keep screen flow separate from gameplay logic.
- Avoid hidden coupling between systems.

## 4. State Model
### Game States
- idle
- aiming
- flying
- scoring
- results

### Persistent State
- current score
- arrows remaining
- current round
- high score
- last result

### Transient State
- pull distance
- aim offset
- arrow position
- arrow velocity
- hit/miss feedback
- floating score effects

## 5. Gameplay Model
### Input
- Pointer down starts a pull.
- Pointer move updates draw distance and aim offset.
- Pointer up fires the arrow.

### Arrow Flight
- The arrow follows a precomputed or continuously simulated trajectory.
- Gravity should be subtle and arcade-friendly.
- The shot should feel responsive, not heavy.

### Collision
- Hit detection should happen at the target plane.
- Use screen-space collision against rendered target geometry.
- Map impact distance to a scoring ring.

### Scoring
- 10 / 7 / 5 / 3 points by ring
- Update score immediately on hit
- Emit a result event for feedback effects

## 6. Rendering Rules
### Canvas Only for Gameplay
Gameplay visuals should be rendered on Canvas 2D.

### UI Layer
Menus and results can be DOM overlays if they simplify implementation.

### Visual Stack
Recommended draw order:
1. background
2. ground / parallax environment
3. target
4. arrow in flight
5. bow / foreground
6. HUD / overlays
7. hit effects

## 7. Input Rules
- Touch devices must not scroll the page during a shot.
- Input should be tolerant of accidental movement.
- Pull gesture must have a dead zone before power registers.
- Horizontal adjustment should be clamped and minimal.
- Release must fire on the same frame or as close as possible.

## 8. Data Persistence
### Storage
Use LocalStorage only.

### Stored Values
- `highScore`
- optional `settings` if added later

### Rules
- Reads must be safe if storage is empty.
- Writes must not block gameplay.
- Storage format should be versionable.

## 9. Performance Requirements
- Stable 60 FPS target
- Minimal garbage creation during gameplay
- No expensive layout thrashing in the game loop
- Avoid per-frame DOM updates for gameplay state

## 10. Debugging Requirements
The codebase should expose clear logging or debug hooks for:
- input state,
- arrow path,
- collision result,
- score updates,
- state transitions.

## 11. Testing Requirements
### Manual Tests
- mobile drag and release
- miss shot
- bullseye hit
- results screen
- reload after high score save

### Edge Cases
- very short drag
- very long drag
- pointer cancel
- orientation resize
- rapid retries

## 12. Maintainability Rules
- Use explicit types.
- Keep system interfaces small.
- Avoid over-abstracting early.
- Put magic numbers in constants.
- Keep formulas readable and documented.

## 13. Out of Scope for MVP
- backend APIs
- multiplayer sync
- server persistence
- cloud saves
- matchmaking
- anti-cheat
- analytics pipeline

## 14. Technical Exit Criteria
The implementation is ready when:
- gameplay is deterministic enough to debug,
- input feels responsive,
- hits match what the player sees,
- and the app performs well on mobile browsers.
