# Requirements: Archery Game

**Defined:** 2026-06-08
**Core Value:** Deliver a polished single-player archery game that opens instantly in a mobile browser, teaches itself through interaction, rewards accuracy with clear feedback, and can be finished in short sessions.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Main Menu

- [ ] **MENU-01**: Game shows a main menu with title and Play button on launch
- [ ] **MENU-02**: Main menu displays current high score
- [ ] **MENU-03**: Main menu uses simple fade transition to gameplay

### Gameplay — Shooting

- [ ] **SHOT-01**: Player can drag to pull the bow string
- [ ] **SHOT-02**: Bow pull visibly increases string tension
- [ ] **SHOT-03**: Player releases to fire arrow
- [ ] **SHOT-04**: Arrow leaves from the nock point cleanly
- [ ] **SHOT-05**: Arrow follows a parabolic trajectory with subtle gravity
- [ ] **SHOT-06**: Input works on both touch and mouse devices
- [ ] **SHOT-07**: Touch devices do not scroll the page during a shot
- [ ] **SHOT-08**: Pull gesture has a dead zone before power registers
- [ ] **SHOT-09**: Pointer cancel does not fire a phantom shot
- [ ] **SHOT-10**: Input is tolerant of accidental movement

### Gameplay — Target and Scoring

- [ ] **TARG-01**: Target is rendered with four clearly separated scoring rings
- [ ] **TARG-02**: Scoring rings: Bullseye=10 (gold), Inner=7 (red), Middle=5 (blue), Outer=3 (white)
- [ ] **TARG-03**: Hit detection maps impact position to correct scoring ring
- [ ] **TARG-04**: Score updates immediately on hit
- [ ] **TARG-05**: Miss handling is clear and unambiguous
- [ ] **TARG-06**: Round ends after 3 arrows are shot

### Gameplay — Feedback

- [ ] **FX-01**: Arrow trail visible during flight
- [ ] **FX-02**: Hit impact effect (subtle burst)
- [ ] **FX-03**: Floating score text appears on hit
- [ ] **FX-04**: Brief hit-stop on impact
- [ ] **FX-05**: Miss state is clearly communicated

### HUD

- [ ] **HUD-01**: Score displayed during gameplay with large readable numerals
- [ ] **HUD-02**: Arrows remaining indicator shown
- [ ] **HUD-03**: HUD does not cover the target center or aim line

### Results

- [ ] **RES-01**: Results screen shows final score after round ends
- [ ] **RES-02**: Results screen shows high score with new-high-score highlight
- [ ] **RES-03**: Results screen offers Restart button
- [ ] **RES-04**: Results screen offers Menu button
- [ ] **RES-05**: Quick transition to results (fade or slide)

### Persistence

- [ ] **PERS-01**: High score stored locally via LocalStorage
- [ ] **PERS-02**: High score restored on load
- [ ] **PERS-03**: Reads are safe if storage is empty
- [ ] **PERS-04**: Writes do not block gameplay
- [ ] **PERS-05**: LocalStorage failure (e.g., Safari private mode) handled gracefully with in-memory fallback

### Rendering and Performance

- [ ] **REND-01**: Gameplay rendered on HTML5 Canvas 2D
- [ ] **REND-02**: Canvas scales correctly for device pixel ratio (HiDPI/Retina)
- [ ] **REND-03**: Stable 60 FPS on target mobile browsers
- [ ] **REND-04**: Responsive canvas sizing works on orientation change
- [ ] **REND-05**: Minimal garbage creation during gameplay (object pooling)
- [ ] **REND-06**: Render pipeline: background → ground → target → arrow → bow → HUD → effects

### App Shell

- [ ] **SHELL-01**: Game works offline after first load (service worker/PWA)
- [ ] **SHELL-02**: Safe-area-inset handling for iOS notch/Dynamic Island
- [ ] **SHELL-03**: Viewport configured correctly (viewport-fit=cover)
- [ ] **SHELL-04**: No dependency on network connectivity after initial load

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Post-MVP Enhancement

- **WIND-01**: Wind mechanic affects arrow trajectory
- **POWR-01**: Power meter for shot strength visualization
- **SOUND-01**: Sound effects for draw, release, hit, miss
- **MODE-01**: Multiple game modes (e.g., Classic Duel, Endless)
- **PROF-01**: Player progression or stats tracking
- **SKIN-01**: Target/bow cosmetic variations

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Multiplayer | Not core to single-player value |
| Accounts or login | No backend dependency for MVP |
| Server/backend | Offline-first design |
| Leaderboards | Requires backend; defer to v2+ |
| Economy/shop/upgrades | Scope creep; distracts from core loop |
| Ads | Ship polished first; monetize later |
| Character selection/skins | Deferred post-MVP polish |
| Long-form progression | Short-session design philosophy |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| MENU-01 | Phase 1 — Project Skeleton | Pending |
| MENU-02 | Phase 6 — Screens + Persistence | Pending |
| MENU-03 | Phase 6 — Screens + Persistence | Pending |
| SHOT-01 | Phase 2 — Input + Bow | Pending |
| SHOT-02 | Phase 2 — Input + Bow | Pending |
| SHOT-03 | Phase 2 — Input + Bow | Pending |
| SHOT-04 | Phase 2 — Input + Bow | Pending |
| SHOT-05 | Phase 3 — Target + Shooting | Pending |
| SHOT-06 | Phase 2 — Input + Bow | Pending |
| SHOT-07 | Phase 2 — Input + Bow | Pending |
| SHOT-08 | Phase 2 — Input + Bow | Pending |
| SHOT-09 | Phase 2 — Input + Bow | Pending |
| SHOT-10 | Phase 2 — Input + Bow | Pending |
| TARG-01 | Phase 3 — Target + Shooting | Pending |
| TARG-02 | Phase 3 — Target + Shooting | Pending |
| TARG-03 | Phase 4 — Hit Detection + Scoring | Pending |
| TARG-04 | Phase 4 — Hit Detection + Scoring | Pending |
| TARG-05 | Phase 4 — Hit Detection + Scoring | Pending |
| TARG-06 | Phase 4 — Hit Detection + Scoring | Pending |
| FX-01 | Phase 5 — Visual Feedback | Pending |
| FX-02 | Phase 5 — Visual Feedback | Pending |
| FX-03 | Phase 5 — Visual Feedback | Pending |
| FX-04 | Phase 5 — Visual Feedback | Pending |
| FX-05 | Phase 5 — Visual Feedback | Pending |
| HUD-01 | Phase 4 — Hit Detection + Scoring | Pending |
| HUD-02 | Phase 4 — Hit Detection + Scoring | Pending |
| HUD-03 | Phase 4 — Hit Detection + Scoring | Pending |
| RES-01 | Phase 6 — Screens + Persistence | Pending |
| RES-02 | Phase 6 — Screens + Persistence | Pending |
| RES-03 | Phase 6 — Screens + Persistence | Pending |
| RES-04 | Phase 6 — Screens + Persistence | Pending |
| RES-05 | Phase 6 — Screens + Persistence | Pending |
| PERS-01 | Phase 6 — Screens + Persistence | Pending |
| PERS-02 | Phase 6 — Screens + Persistence | Pending |
| PERS-03 | Phase 6 — Screens + Persistence | Pending |
| PERS-04 | Phase 6 — Screens + Persistence | Pending |
| PERS-05 | Phase 6 — Screens + Persistence | Pending |
| REND-01 | Phase 1 — Project Skeleton | Pending |
| REND-02 | Phase 1 — Project Skeleton | Pending |
| REND-03 | Phase 6 — Screens + Persistence | Pending |
| REND-04 | Phase 1 — Project Skeleton | Pending |
| REND-05 | Phase 5 — Visual Feedback | Pending |
| REND-06 | Phase 1 — Project Skeleton | Pending |
| SHELL-01 | Phase 1 — Project Skeleton | Pending |
| SHELL-02 | Phase 1 — Project Skeleton | Pending |
| SHELL-03 | Phase 1 — Project Skeleton | Pending |
| SHELL-04 | Phase 1 — Project Skeleton | Pending |

**Coverage:**
- v1 requirements: 47 total
- Mapped to phases: 47
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-08*
*Last updated: 2026-06-08 after initial definition*
