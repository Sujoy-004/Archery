# Archery Game

## What This Is

A mobile-first, offline archery mini-game where players drag-to-pull a bow and release to shoot arrows at a stationary target. Built for casual mobile players who want a quick, satisfying skill-based arcade loop that opens instantly in a browser and teaches itself through interaction.

## Core Value

Deliver a polished single-player archery game that opens instantly in a mobile browser, teaches itself through interaction, rewards accuracy with clear feedback, and can be finished in short sessions.

## Requirements

### Validated

(Not yet — ship to validate)

### Active

- [ ] **AUTH-01**: Player can start a game from the main menu with one tap
- [ ] **AUTH-02**: Game teaches drag-to-release mechanic naturally (no tutorial)
- [ ] **AUTH-03**: Player can shoot arrows at a stationary target
- [ ] **AUTH-04**: Target has four scoring rings (Bullseye=10, Inner=7, Middle=5, Outer=3)
- [ ] **AUTH-05**: Score updates immediately on hit with visual feedback
- [ ] **AUTH-06**: Round ends after fixed number of arrows (3)
- [ ] **AUTH-07**: Results screen shows final score, high score, restart, and menu
- [ ] **AUTH-08**: High score persists locally via LocalStorage
- [ ] **AUTH-09**: Game runs on mobile browsers at 60 FPS
- [ ] **AUTH-10**: No browser scrolling during gameplay
- [ ] **AUTH-11**: Canvas 2D rendering for gameplay; DOM overlays for UI
- [ ] **AUTH-12**: Visual effects: arrow trail, hit impact, floating score text

### Out of Scope

- Multiplayer — not core to single-player value
- Accounts or login — no backend dependency
- Server/backend — offline-first
- Leaderboards — deferred to v2
- Economy/shop/upgrades — scope creep for MVP
- Wind mechanics — adds complexity without proportional fun
- Character selection/skins — deferred
- Long-form progression — short-session design
- Ads in MVP — ship polished, monetize later

## Context

Built as a lightweight browser game inspired by casual local-multiplayer archery modes. The target audience is casual mobile players looking for quick time-pass games. The experience should feel polished and arcade-like rather than simulation-heavy. Sound is reserved for a future update.

## Constraints

- **Platform**: Mobile web first, desktop compatible
- **Rendering**: HTML5 Canvas 2D (no WebGL)
- **Persistence**: LocalStorage only, no backend
- **Performance**: Stable 60 FPS on low-end mobile devices
- **Offline**: Must work offline after first load
- **Input**: Touch-first, drag-to-pull, no keyboard required

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Canvas 2D over DOM/WebGL | Simplest performant option for 2D archery | — Pending |
| No backend for MVP | Single-player offline doesn't need server | — Pending |
| 3 arrows per round | Short session constraint (under 2 min) | — Pending |
| 4 scoring rings (10/7/5/3) | Simple, readable scoring without complexity | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-08 after initialization*
