# MVP

## Product Summary
Build a mobile-first, offline archery mini-game inspired by the archery mode in a casual local-multiplayer game collection. The MVP should feel instantly understandable, quick to play, and satisfying to repeat.

## Product Goal
Deliver a polished single-player archery game that:
- opens instantly in a mobile browser,
- teaches itself through interaction,
- rewards accuracy with clear feedback,
- and can be finished in short sessions.

## Platform
- Mobile web app first
- Desktop browser compatible
- Offline-capable after first load

## Core Gameplay
- Player controls a bow and shoots arrows at a stationary target.
- Each shot is created by drag-to-pull and release.
- Target contains four scoring rings.
- Score is assigned by ring hit.
- The player completes a short round, then sees results.

## MVP Scope
### In Scope
- Single-player mode
- Canvas 2D rendering
- Drag / pull / release shooting
- Static target with scoring rings
- Round-based scoring
- Results screen
- Local high score persistence
- Basic visual effects: trail, impact feedback, floating score

### Out of Scope
- Multiplayer
- Accounts or login
- Server/backend
- Leaderboards
- Economy / shop / upgrades
- Wind mechanics
- Character selection
- Skins
- Long-form progression
- Ads in the MVP

## Gameplay Contract
- The game must be understandable within the first shot.
- A player should never need a tutorial to discover how to fire.
- Shot feedback must be immediate and readable.
- Difficulty should come from distance, timing, and target size only.
- No hidden randomness that makes the game feel unfair.

## Success Criteria
The MVP is successful if:
1. The player can understand how to shoot without explanation.
2. A full session can be completed in under 2 minutes.
3. Scoring is obvious and trustworthy.
4. The game feels good on a low-end phone.
5. A player wants to restart immediately after finishing.

## Definition of Done
The MVP is done when:
- launch to menu works,
- play loop works,
- scoring works,
- results work,
- high score persists locally,
- and the game runs smoothly on mobile browsers.
