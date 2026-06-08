# Roadmap: Archery Game

**Created:** 2026-06-08
**Granularity:** Standard (5-8 phases)
**Project Mode:** MVP

---

## Core Value

Deliver a polished single-player archery game that opens instantly in a mobile browser, teaches itself through interaction, rewards accuracy with clear feedback, and can be finished in short sessions.

---

## Phases

- [ ] **Phase 1: Project Skeleton** — Vite + TypeScript scaffold, Canvas bootstrap, game loop, state manager, responsive sizing, DPR scaling, viewport/safe-area config, offline PWA shell
- [ ] **Phase 2: Input + Bow** — Pointer Events unification, drag-to-pull bow mechanic, string tension rendering, touch scroll prevention, dead zone, pointer cancel handling
- [ ] **Phase 3: Target + Shooting** — 4-ring target rendering, arrow trajectory physics with arcade gravity, arrow entity and render layer
- [ ] **Phase 4: Hit Detection + Scoring** — Collision system, ring mapping, score tracking, miss handling, round end logic, HUD with score and arrows remaining
- [ ] **Phase 5: Visual Feedback** — Arrow trail during flight, hit impact particles, floating score text, hit-stop, miss communication, object pooling for effects
- [ ] **Phase 6: Screens + Persistence** — Main menu with high score, results screen, screen transitions, LocalStorage high-score persistence, Safari private mode fallback, 60 FPS performance guarantee

---

## Phase Details

### Phase 1: Project Skeleton
**Goal:** Foundational project structure boots and renders a blank canvas at 60 FPS with responsive sizing, DPR scaling, viewport/safe-area handling, and offline PWA shell
**Mode:** mvp
**Depends on:** Nothing
**Requirements:** MENU-01, REND-01, REND-02, REND-04, REND-06, SHELL-01, SHELL-02, SHELL-03, SHELL-04
**Success Criteria** (what must be TRUE):
  1. Game boots and shows a main menu with title and Play button on first launch
  2. Canvas renders at correct resolution on both standard and HiDPI/Retina displays (no blur)
  3. Canvas resizes correctly on orientation change and window resize
  4. Game loads and runs with no network after initial load (offline PWA)
  5. Content respects iOS safe-area-inset (notch, Dynamic Island) with no clipping
**Plans:** TBD
**UI hint**: yes

### Phase 2: Input + Bow
**Goal:** Player can touch/drag the bow string with natural feel, see tension visual feedback, and release cleanly — no phantom shots, no page scrolling
**Mode:** mvp
**Depends on:** Phase 1
**Requirements:** SHOT-01, SHOT-02, SHOT-03, SHOT-04, SHOT-06, SHOT-07, SHOT-08, SHOT-09, SHOT-10
**Success Criteria** (what must be TRUE):
  1. Player can drag on the bow string and see it deform proportionally to pull distance
  2. Player can release to fire the arrow cleanly from the nock point
  3. Touch input does not cause page scroll or zoom during the shot gesture
  4. Pull gesture has a dead zone — accidental taps do not register as shots
  5. Pointer cancel/interruption does not fire a phantom shot
**Plans:** TBD

### Phase 3: Target + Shooting
**Goal:** A stationary 4-ring scoring target is visible on canvas, and arrows fly with a parabolic trajectory when released
**Mode:** mvp
**Depends on:** Phase 2
**Requirements:** TARG-01, TARG-02, SHOT-05
**Success Criteria** (what must be TRUE):
  1. Target appears with four clearly separated scoring rings (Bullseye=10 gold, Inner=7 red, Middle=5 blue, Outer=3 white)
  2. Arrow follows a visible parabolic arc with subtle gravity after release
  3. Arrow rendering is smooth and visible across the full flight path
**Plans:** TBD

### Phase 4: Hit Detection + Scoring
**Goal:** Arrow impact is detected against the target rings, score is calculated and displayed on a HUD, round ends after 3 arrows
**Mode:** mvp
**Depends on:** Phase 3
**Requirements:** TARG-03, TARG-04, TARG-05, TARG-06, HUD-01, HUD-02, HUD-03
**Success Criteria** (what must be TRUE):
  1. Arrow impact position maps to correct scoring ring (10/7/5/3) and score updates immediately
  2. Miss (arrow landing outside all rings) is clearly communicated — no ambiguity
  3. Round ends after exactly 3 arrows have been shot
  4. Score is displayed with large readable numerals during gameplay
  5. Arrows-remaining indicator is visible and does not cover target center or aim line
**Plans:** TBD

### Phase 5: Visual Feedback
**Goal:** Every shot produces satisfying visual feedback — arrow trail, hit impact burst, floating score text, and hit-stop — all without GC stutter
**Mode:** mvp
**Depends on:** Phase 4
**Requirements:** FX-01, FX-02, FX-03, FX-04, FX-05, REND-05
**Success Criteria** (what must be TRUE):
  1. Arrow leaves a visible trail during flight
  2. Hit impact shows a subtle particle burst at the impact point
  3. Floating score text rises from the hit point showing the ring value
  4. Brief hit-stop (pause) occurs on impact for satisfying feel
  5. Miss (no rings hit) is clearly communicated with distinct visual feedback
**Plans:** TBD

### Phase 6: Screens + Persistence
**Goal:** Full game flow — main menu → gameplay → results screen → replay or menu — with high score persisting across sessions and stable 60 FPS
**Mode:** mvp
**Depends on:** Phase 5
**Requirements:** MENU-02, MENU-03, RES-01, RES-02, RES-03, RES-04, RES-05, PERS-01, PERS-02, PERS-03, PERS-04, PERS-05, REND-03
**Success Criteria** (what must be TRUE):
  1. Main menu displays current high score below the title
  2. Main menu uses a fade transition into gameplay
  3. Results screen shows final score, high score (with highlight if new), Restart button, and Menu button
  4. High score persists across browser sessions via LocalStorage and is restored on load
  5. Game runs at stable 60 FPS on target mobile browsers; LocalStorage failure (Safari private mode) is handled gracefully with in-memory fallback
**Plans:** TBD
**UI hint**: yes

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Project Skeleton | 0/0 | Not started | - |
| 2. Input + Bow | 0/0 | Not started | - |
| 3. Target + Shooting | 0/0 | Not started | - |
| 4. Hit Detection + Scoring | 0/0 | Not started | - |
| 5. Visual Feedback | 0/0 | Not started | - |
| 6. Screens + Persistence | 0/0 | Not started | - |

---

*Roadmap created: 2026-06-08*
