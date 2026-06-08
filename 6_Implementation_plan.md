# Implementation Plan

## Phase 1 — Project Setup
### Goals
- establish the codebase
- create the canvas render loop
- define core types and constants

### Tasks
- initialize Vite + TypeScript
- configure linting / formatting
- create base canvas bootstrap
- add responsive sizing

### Exit Criteria
- blank app runs on desktop and mobile
- canvas scales correctly
- project builds cleanly

## Phase 2 — Core Input and Shooting
### Goals
- make the bow usable
- make shooting feel immediate

### Tasks
- build drag-to-pull input
- compute pull strength
- release to fire
- support mobile touch behavior

### Exit Criteria
- player can shoot an arrow reliably
- input never triggers accidental page scroll

## Phase 3 — Target and Scoring
### Goals
- add gameplay meaning

### Tasks
- render target
- implement scoring rings
- add hit detection
- update score on hit
- add miss handling

### Exit Criteria
- points are assigned correctly
- hits and misses are visually understandable

## Phase 4 — Round Flow and Results
### Goals
- turn shots into a complete session

### Tasks
- track arrows remaining
- end the round
- show results screen
- add restart and menu actions

### Exit Criteria
- full play loop is complete from start to finish

## Phase 5 — Feedback and Polish
### Goals
- make the game feel worth playing repeatedly

### Tasks
- add floating score text
- add subtle hit-stop
- add trail and impact effects
- polish HUD and transitions

### Exit Criteria
- the game feels responsive and satisfying

## Phase 6 — Persistence
### Goals
- preserve player progress locally

### Tasks
- store high score in LocalStorage
- restore high score on load
- handle missing storage safely

### Exit Criteria
- high score persists across sessions

## Phase 7 — QA and Release
### Goals
- ship a stable build

### Tasks
- mobile device testing
- performance verification
- layout checks
- bug fixing
- final build verification

### Exit Criteria
- no major usability bugs
- stable on target devices
- ready for deployment

## Delivery Rule
Do not expand scope until the current phase is fully verified.
