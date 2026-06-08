# Product Requirements Document

## 1. Product Overview
A lightweight archery game for casual mobile players. The experience is simple, fast, and replayable. It is designed as a browser game with no backend dependency.

## 2. Target Users
Primary users:
- casual mobile players,
- people looking for a quick time-pass game,
- players who enjoy short skill-based arcade loops.

Secondary users:
- desktop players,
- offline-first users,
- players on low-end Android devices.

## 3. User Needs
Users need a game that:
- starts fast,
- is easy to understand,
- gives immediate feedback,
- is fair,
- and can be replayed repeatedly without friction.

## 4. Core User Stories
### US-1 Start a game
As a player, I want to press play and begin shooting immediately.

### US-2 Learn the controls naturally
As a player, I want the game to make the drag-and-release mechanic obvious without instructions.

### US-3 Score points
As a player, I want to see where my arrow landed and how many points I earned.

### US-4 Finish a round
As a player, I want the game to end cleanly and show my final score.

### US-5 Replay
As a player, I want to restart quickly and try to beat my previous score.

### US-6 Retain a high score
As a player, I want my best score saved locally so it persists between sessions.

## 5. Functional Requirements
### FR-1 Main Menu
The game must show:
- title,
- Play button,
- high score.

### FR-2 Gameplay Screen
The game must show:
- bow,
- target,
- score,
- arrows remaining,
- clear shot feedback.

### FR-3 Shooting
The player must be able to:
- drag to pull,
- release to shoot,
- see the shot travel,
- and receive an immediate result.

### FR-4 Scoring
The target must contain four scoring zones:
- Bullseye = 10
- Inner ring = 7
- Middle ring = 5
- Outer ring = 3

### FR-5 Round Structure
Each round must have a fixed number of arrows. The recommended MVP default is 3 arrows per round.

### FR-6 Results
At the end of the round the game must show:
- final score,
- arrows shot,
- high score indicator,
- restart and menu actions.

### FR-7 Persistence
The best score must be stored locally using LocalStorage.

## 6. Non-Functional Requirements
### Performance
- Target 60 FPS on typical mobile browsers.
- Avoid heavy DOM animation.
- Keep the render pipeline simple.

### Reliability
- No crashes on refresh.
- No dependency on network connectivity.
- Input should not break when the browser handles touch gestures.

### Usability
- The first shot should teach the mechanic.
- Visual feedback must make hits and misses obvious.
- The interface must remain readable on small screens.

## 7. Product Constraints
- No backend for MVP.
- No online multiplayer.
- No account system.
- No complex economy.
- No feature bloat.

## 8. Acceptance Criteria
The product is acceptable when:
- a user can play end-to-end without confusion,
- scoring is trustworthy,
- results are displayed correctly,
- high score persists locally,
- and the experience feels polished enough to be shippable.

## 9. Risks
- Controls may feel unclear if pull feedback is weak.
- Scoring may feel arbitrary if hit detection is imprecise.
- Mobile browser gesture handling may interfere with touch input.
- Performance may suffer if the render stack becomes too layered.

## 10. MVP Positioning
This is a premium-feeling casual arcade game, not a simulation. The product should feel polished, predictable, and satisfying rather than technically complex.
