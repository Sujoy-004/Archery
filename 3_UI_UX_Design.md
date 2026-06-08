# UI / UX Design

## 1. Design Goals
- Immediate clarity
- Touch-first interaction
- High contrast
- Low visual noise
- Fast feedback
- Arcade feel, not simulation feel

## 2. Visual Style
### Direction
- clean
- colorful
- readable
- lightly stylized
- minimal realism

### Avoid
- cluttered overlays
- tiny text
- decorative UI that competes with gameplay
- low-contrast targets
- complex animation timing

## 3. Layout Principles
- Keep primary gameplay centered.
- Keep the target visually dominant.
- Keep HUD information above or away from the shot path.
- Leave enough empty space for the arrow trajectory to read clearly.
- Ensure all important UI works on narrow mobile screens.

## 4. Screen Set
### Main Menu
Contains:
- title
- play button
- current high score
- optional small subtitle

Behavior:
- simple fade in
- no unnecessary motion

### Gameplay Screen
Contains:
- target
- bow
- arrows remaining
- score
- optional shot feedback

Behavior:
- game area should be immediately legible
- HUD should not cover the target or aim line

### Results Screen
Contains:
- final score
- high score state
- restart button
- menu button

Behavior:
- quick transition
- strong reward feedback for new highs

## 5. HUD Design
### Recommended Elements
- score
- arrows remaining
- round indicator
- optional progress indicator

### Rules
- use large readable numerals
- keep labels short
- avoid placing HUD over the target center
- update values with small motion or color emphasis, not heavy animation

## 6. Target Design
### Target Rules
- clearly separated scoring rings
- obvious center
- enough contrast between rings
- readable at small screen sizes
- no ambiguity on hit location

### Scoring Colors
- Bullseye: gold
- Inner ring: red
- Middle ring: blue
- Outer ring: white or light neutral

## 7. Bow Design
### Requirements
- visible from the start
- obviously interactive
- large enough to understand on mobile
- visually tied to the player side of the screen

### Behavior
- bow pull should visibly increase string tension
- released arrow should leave from the nock point cleanly

## 8. Motion Design
### Good Motion
- hit-stop on impact
- brief score popup
- subtle arrow trail
- short result transition

### Avoid
- long screen shakes
- overcomplicated bounce effects
- motion that hides gameplay feedback

## 9. Feedback Design
### Hit
- score popup
- subtle burst
- slight pause
- sound slot reserved for later if needed

### Miss
- clear miss state
- quick reset or fade
- no confusion about whether the shot counted

### New High Score
- visible highlight on result screen
- optional small celebratory effect
- clear label showing the new best score

## 10. Typography
- Use a clean sans-serif font.
- Prioritize legibility over personality.
- Use bold weight for score and result values.
- Keep labels short.

## 11. Accessibility
- High contrast between UI and background.
- Avoid color-only communication where possible.
- Keep tap targets large enough for thumbs.
- Keep text readable in portrait mode.

## 12. Mobile UX Rules
- No browser scrolling during gameplay.
- No controls smaller than a thumb-friendly size.
- No interaction requiring precision beyond the game itself.
- Game should still work if the browser chrome changes height.

## 13. Copy Tone
Use short, direct UI copy:
- Play
- Restart
- Menu
- Score
- High Score
- Arrows Left

Avoid verbose tooltips.

## 14. Visual Hierarchy
Priority order:
1. target
2. arrow / hit feedback
3. score
4. arrows remaining
5. menu controls

## 15. Deliverable Standard
The UI should be good enough to ship as a paid casual browser game:
- polished
- readable
- consistent
- minimally distracting
- satisfying on repeated plays
