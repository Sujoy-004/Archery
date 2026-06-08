# Feature Research

**Domain:** Mobile-first casual archery arcade game (single-player, browser-based)
**Researched:** 2026-06-08
**Confidence:** HIGH

## Feature Landscape

Based on analysis of 15+ mobile/browser archery games (Archery World Tour, Archery Master, Archery King, Stickman Archer, Archery Blast, Archery Elite, Tiny Archer, Bow And Arrow, Forest Fortune, Archery Champ, Archer Arena, Elite Archery, King Archer, Archery Clash, World Archery League) across HTML5 browser and native mobile platforms.

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete or broken.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Drag-to-pull-and-release shooting | Universal standard across ALL mobile archery games. This is THE control scheme. Any alternative (tap-to-shoot, button-based) feels wrong. | MEDIUM | Touch down starts aim, drag back sets angle+power, release fires. Must feel responsive — input latency kills the feel. |
| Ring-based target with scoring | 100% of surveyed games use concentric ring targets. Standard: Bullseye (10), Inner (7), Middle (5), Outer (3). Colors typically yellow/red/blue/white. | LOW | 4 rings is standard. Fewer feels shallow, more is noisy. The 10/7/5/3 pattern from PROJECT.md is well-calibrated. |
| Immediate shot feedback | Arrow must visibly stick in the target at exact hit location. Score must appear on hit. Delay breaks trust. | MEDIUM | Three feedback layers: (1) arrow trail during flight, (2) impact animation/arrow embed, (3) floating score text. All three are table stakes. |
| Results screen after round | Shows final score, high score comparison, arrows shot, restart + menu buttons. Every game has this. | LOW | Must appear immediately when round ends. No animation delays. Restart should be the most prominent button. |
| High score persistence via LocalStorage | Users expect their best score to survive browser close. Every HTML5 archery game implements this. | LOW | Standard pattern: save on round end, read on menu load. Zero dependencies needed. |
| Simple main menu | Title + Play button + high score display. One tap to start. No splash screens, no loading bars. | LOW | The "Tap to Start" or single Play button pattern is universal in casual browser games. |
| Arrow count remaining display | Players need to know how many shots left in the round. Shown as icons (arrow pictograms) or "3/3" counter. | LOW | Best done as DOM overlay (per ARCHITECTURE.md constraint). Position in corner, always visible. |
| Arrow trail during flight | A visual trail (particle, line, or glow) behind the arrow as it travels. Without it, the shot feels invisible mid-flight. | LOW | Simple implementation: short fading line or particle burst behind arrow position each frame. |
| Float score text on hit | Score number that rises and fades from hit location. Communicates "what you got" instantly without looking at UI. | LOW | Classic arcade pattern. Small upward animation + fade out over ~1s. Color-code by ring (gold for 10, etc). |
| No scrolling/zooming during gameplay | Touch input must not trigger browser gestures. Miss this and the game is unplayable on mobile. | LOW | `touch-action: none` on canvas, prevent default on touch events. Standard mobile web pattern. |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valuable for distinction.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Wind indicator + physics** | Transforms shooting from "aim at center" into a reading+compensation skill puzzle. Archery World Tour and Archery Master both credit wind as their core depth mechanic. | MEDIUM | Wind direction + strength shown via arrow/compass indicator. Arrow is pushed off course proportionally during flight. Changes each shot or every few shots. |
| **Power/draw strength meter** | Shows how far the bow is pulled back, giving player explicit feedback on shot power. Stickman Archer uses a color-coded bar (red/yellow/green). Without this, players guess. | LOW | Bar fills/inverts as drag distance increases. Green zone = optimal, red = over-pull. Can be combined with trajectory arc. |
| **Multiple game modes** | Classic (fixed arrows), Balloon Challenge (one-arrow-per-target, miss=end), Moving Targets, Practice (unlimited arrows). Each adds replayability. | HIGH | Each mode needs its own logic, UI, and balancing. Balloon mode is particularly popular (Archery World Tour, Archery Blast). Start with one mode (Classic), add others post-launch. |
| **Progressive difficulty levels** | Multiple levels with increasing target distance, smaller rings, and environmental complexity. Star rating per level (1-3 stars). Archery World Tour has 50 levels. | HIGH | Requires level configuration system, difficulty curve design, and unlock tracking. Scope risk for MVP. |
| **Moving targets** | Targets that slide horizontally, swing, or rotate. Adds timing to precision. Present in Archery World Tour, Elite Archery, Archery Blast. | MEDIUM | Predictable patterns (oscillating, linear). Player must lead the target. Can be introduced in later levels only. |
| **Sound effects** | Bow draw creak, release twang, arrow whoosh, hit thud, miss rustle, score chime. Massively increases polish feel. | LOW | HTML5 Web Audio API. Short synthesized or sampled sounds. No streaming. Toggle on/off control. |
| **Trajectory prediction line** | Dotted arc showing predicted arrow path before release. Stickman Archer uses white dotted line + power meter. Reduces guesswork. | MEDIUM | Physics simulation preview drawn on canvas. Must update in real-time as player drags. Performance cost if not optimized. |
| **Instant restart** | Ability to restart the round with one tap from results screen (or mid-round). "One more try" loop is the core retention mechanic. | LOW | Just reset game state and re-initialize. No scene transitions. The gold standard is results → restart in <1 second. |
| **Combo/streak multiplier** | Consecutive good shots (inner rings) build a multiplier. Rewards consistency. Tiny Archer and Archery Master use this. | LOW | Track consecutive hits above threshold; multiply score. Reset on miss or outer ring. Adds tension and reward. |
| **Haptic feedback on mobile** | Short vibration on draw start, stronger vibration on release, success/failure buzz. Stickman Archer implements this. | LOW | Use Vibration API (`navigator.vibrate()`). Subtle — enhances without being required. Gracefully degrade if unsupported. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems for this specific product.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Ads in MVP** | Monetization pressure | Destroys the "polished, premium-feeling" experience. Player reviews consistently cite aggressive ads as the primary reason for uninstalling archery games. | Ship without ads. Add optional rewarded video for extra content later if monetization is needed. |
| **Multiplayer / PvP** | Competitive appeal | Vast scope increase: networking, matchmaking, anti-cheat, sync, real-time physics. Contradicts offline-first constraint. | Single-player with local high score as the "competition." Add async leaderboards v2 if desired. |
| **Login / account system** | Cross-device progress | Requires backend, auth flow, password management, privacy compliance. Massive friction for a casual game. | LocalStorage is sufficient for MVP. v2 cloud sync could use anonymous device IDs. |
| **Pay-to-win upgrades** | Revenue potential | Destroys the core value: pure skill-based scoring. If paid gear gives accuracy advantage, score loses meaning. Multiple negative reviews cite this in Archery Elite. | Cosmetic-only customization (arrow colors, bow skins) if monetization is added later. |
| **Auto-aim / aim assist** | Reducing frustration | Undermines the entire value proposition of skill-based archery. If the game aims for you, hitting bullseye feels hollow. | Make the drag mechanic feel good and precise. Clear visual feedback so misses feel instructive, not punishing. |
| **Complex simulation physics** | Realism appeal | Arrow weight, draw weight, archer stamina, bow maintenance. Too simulation-heavy for casual arcade. Increases dev cost without proportional fun. | Simple gravity + wind is sufficient. The PROJECT.md and PRD both say "arcade not simulation" — honor this. |
| **Forced tutorial / instructions** | Onboarding concern | The drag-to-release mechanic should be self-evident. Tutorial text before first shot creates friction. Negative reviews call it "fussy." | Environmental teaching: first target is very close/easy. Visual hint (arrow animation on bow) if player hesitates >5s. |
| **Timed rounds (default mode)** | Urgency/excitement | Rushing contradicts the precision/calm satisfaction of the genre. Timed modes exist as OPTIONS in some games, never as default. | Fixed-arrow rounds (no timer). Time Attack can be an optional mode added later. |
| **Punishing miss feedback** | Realism / consequence | Negative animations (arrow shattering, "miss" text, sad sound) feel bad for casual players and discourage retries. | Neutral feedback on miss: arrow passes through, subtle "plink" sound. Highlight successes positively. |
| **Economy / shop / currencies** | Engagement / retention | Coins, gems, energy systems create FOMO mechanics that conflict with "pick up and play for 2 minutes." Massive scope. | Pure skill-based loop. If progression is needed later, use score-unlocked content (reach 100pts to unlock new arrow color). |

## Feature Dependencies

```
[Results Screen]
    └──requires──> [Round Logic (3 arrows)]
                       └──requires──> [Shooting Mechanic (drag-release)]
                                          └──requires──> [Touch Input Handling]
                                                              └──requires──> [Canvas 2D Renderer]

[Score Display] ──requires──> [Scoring System (ring detection)]
                                  └──requires──> [Target Rendering + Hit Detection]

[High Score Persistence] ──requires──> [Scoring System]
                                          └──enhances──> [Results Screen]

[Floating Score Text] ──requires──> [Hit Detection]
                                    └──enhances──> [Score Display]

[Arrow Trail] ──requires──> [Arrow Physics/Animation]
                            └──enhances──> [Shot Feedback]

[Wind System]
    └──requires──> [Arrow Physics (drift offset)]
                       └──enhances──> [Core Shooting Mechanic]

[Power Meter] ──enhances──> [Shooting Mechanic]

[Trajectory Prediction]
    └──requires──> [Arrow Physics Simulation]
                       └──enhances──> [Power Meter]

[Combo Multiplier]
    └──requires──> [Scoring System]
                       └──enhances──> [Results Screen]

[Multiple Game Modes]
    └──requires──> [Core Game Loop (shoot → score → results)]
                       └──requires──> [All Table Stakes features]
```

### Dependency Notes

- **[Shooting Mechanic] is the root dependency:** Everything branches from the core drag-to-release loop. Get this right before building anything else. If the shooting doesn't feel good, nothing else matters.
- **[Results Screen] depends on [Round Logic]:** You can't show results without a defined round structure (3 arrows). The round-end trigger must be reliable.
- **[Wind System] enhances [Core Shooting Mechanic]:** Wind is a modifier on the existing physics, not a separate system. It applies a drift offset to the arrow during flight. Easy to add after core shooting is working, but designing the physics to accommodate wind from the start avoids refactoring.
- **[Combo Multiplier] depends on [Scoring System]:** Multipliers are a scoring modifier. Add only after base scoring is verified correct.
- **[Multiple Game Modes] requires ALL table stakes:** Each game mode reuses the core loop but modifies rules (arrow count, target behavior, win condition). Table stakes must be solid before mode branching.

## MVP Definition

### Launch With (v1)

Minimum viable product — what's needed to validate the concept.

- [x] **Drag-to-pull-and-release shooting** — Core mechanic. Non-negotiable.
- [x] **Ring-based target (4 rings, 10/7/5/3)** — Standard scoring. Simple, readable.
- [x] **Immediate shot feedback** — Trail + impact embed + floating score text.
- [x] **3-arrow round structure** — Short session constraint (<2 min per PRD).
- [x] **Results screen** — Final score, high score, restart button, menu button.
- [x] **High score persistence (LocalStorage)** — Survives refresh.
- [x] **Simple main menu** — Title, Play button, high score display.
- [x] **Arrow count display** — Shows remaining arrows during round.
- [x] **Mobile touch controls (no scroll/zoom)** — Essential for playability.
- [x] **Instant restart** — <1 second from results screen to new round.

### Add After Validation (v1.x)

Features to add once core loop is working and fun is confirmed.

- [ ] **Wind indicator + physics** — #1 most impactful depth addition. Transforms "point and click" into "read and compensate."
- [ ] **Power/draw meter** — Obvious UX improvement. Players currently guess pull strength.
- [ ] **Sound effects** — Highest polish-per-effort ratio. Bow twang, hit thud, score chime.
- [ ] **Combo/streak multiplier** — Adds tension and rewards consistency. Simple implementation.

**Trigger for adding:** Core loop is fun (test with real users), no critical bugs, 60 FPS sustained on target devices.

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **Multiple game modes** (Balloon Challenge, Moving Targets, Time Attack) — Significant scope. Start with Classic only.
- [ ] **Progressive levels** (50 levels with 3-star rating) — Requires level editor and difficulty curve design.
- [ ] **Trajectory prediction line** — Medium complexity, nice-to-have. Add if players struggle with aim.
- [ ] **Moving targets** — Adds timing dimension. Introduce as harder levels within a mode.
- [ ] **Haptic feedback** — Device-dependent. Low effort but only noticeable once other polish is done.
- [ ] **Character/bow customization** — Cosmetic only. Defer until core engagement is proven.
- [ ] **Leaderboards** — Requires some server component. Async score comparison could work with simple API.

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Drag-to-pull-and-release shooting | HIGH | MEDIUM | P1 |
| Ring-based target with scoring | HIGH | LOW | P1 |
| Immediate shot feedback (trail + impact + score text) | HIGH | MEDIUM | P1 |
| 3-arrow round structure | HIGH | LOW | P1 |
| Results screen | HIGH | LOW | P1 |
| High score persistence | HIGH | LOW | P1 |
| Simple main menu | HIGH | LOW | P1 |
| Arrow count display | HIGH | LOW | P1 |
| No scroll/zoom on touch | HIGH | LOW | P1 |
| Instant restart | MEDIUM | LOW | P1 |
| Wind indicator + physics | HIGH | MEDIUM | P2 |
| Power/draw meter | MEDIUM | LOW | P2 |
| Sound effects | MEDIUM | LOW | P2 |
| Combo/streak multiplier | MEDIUM | LOW | P2 |
| Multiple game modes | HIGH | HIGH | P3 |
| Progressive levels (50+) | MEDIUM | HIGH | P3 |
| Trajectory prediction line | MEDIUM | MEDIUM | P3 |
| Moving targets | MEDIUM | MEDIUM | P3 |
| Haptic feedback | LOW | LOW | P3 |
| Customization (cosmetic) | LOW | MEDIUM | P3 |
| Leaderboards | MEDIUM | MEDIUM | P3 |

**Priority key:**
- **P1:** Must have for launch. Product is broken without it.
- **P2:** Should have, add when core loop is validated. High value-to-cost ratio.
- **P3:** Nice to have, only after product-market fit is established.

## Competitor Feature Analysis

| Feature | Archery World Tour (Famobi) | Archery Master (SUN.STUDIO) | Our Approach |
|---------|----------------------------|------------------------------|--------------|
| **Shooting mechanic** | Drag-to-pull-and-release | Drag-to-pull-and-release | Same — universal standard |
| **Wind physics** | Full wind simulation, changes every shot | Wind indicator, varies by level | Add P2 after core validated |
| **Target types** | Static, moving, rotating, obscured | Static, moving, varying distance | Static first, moving later |
| **Scoring** | Ring-based + star rating (1-3) per level | Ring-based, score per level | 10/7/5/3 rings (simpler) |
| **Round structure** | Variable arrows based on level target | Limited arrows per level | Fixed 3 arrows (simpler, snackable) |
| **Game modes** | World Tour (50 levels) + Balloon Challenge | Classic, Moving, Long Shot, Time Attack | Single Classic mode for MVP |
| **High score** | Per-level best + overall | Leaderboard-based | LocalStorage best score |
| **Power meter** | Not visible (aim crosshair only) | Not visible | Add P2 — significant UX win |
| **Sound** | Yes — bow, hit, ambient | Yes — bow, hit, score | Add P2 — high polish per effort |
| **Monetization** | Ad-supported (app version) | Aggressive ads + IAP | None in MVP — pure experience |
| **Offline** | Yes (after first load) | Yes (HTML5 cached) | Yes — core requirement |
| **Mobile touch** | Yes — full touch support | Yes — full touch support | Yes — core requirement |

## Sources

- **Competitor products analyzed:** Archery World Tour (Famobi/Happylander), Archery Master (SUN.STUDIO), Archery King (Code This Lab), Archery Elite (707 INTERACTIVE), Archery Blast, Stickman Archer (nitinya9av — open source HTML5), Tiny Archer, Bow And Arrow (multiple publishers), Forest Fortune (Ringline), Archery Champ, Archer Arena, Elite Archery, King Archer, Archery Clash, World Archery League (mobirix). Analyzed via Google Play Store listings, App Store pages, HTML5 game portals (Coolmath, Arcadino, MinigamesVille, ZinGames, Kiz10, Volty Games), and developer blogs.
- **Open source references:** nitinya9av/stickman-archer (GitHub), hassanhaseen/ArcheryGame (GitHub), Adnan-Toky/archery-game (GitHub), rgliever/ArcheryCanvasGame (GitHub)
- **Industry patterns:** HTML5 game portals (Famobi distribution model), touch-input patterns for canvas games, arcade scoring feedback loops
- **User sentiment sources:** App Store reviews, Google Play reviews, game portal user comments, developer case studies (Technoloader Archer Arena case study)

---
*Feature research for: Mobile-first casual archery arcade game*
*Researched: 2026-06-08*
