# App Flow

## 1. High-Level Flow
Launch
→ Main Menu
→ Gameplay
→ Results
→ Main Menu

## 2. State Machine
### Idle
- App loads
- Assets initialize
- High score is read from storage

### Main Menu
- User sees title and play entry point
- User can start a session

### Gameplay
- Player shoots arrows
- Score updates in real time
- Round ends after all arrows are used

### Results
- Final score is shown
- High score is compared and updated if needed
- Player chooses restart or return to menu

## 3. Gameplay Sub-Flow
Start Round
→ Aim
→ Pull
→ Release
→ Arrow Flight
→ Hit / Miss
→ Score Feedback
→ Next Arrow
→ Round Complete

## 4. Screen Transitions
Recommended transitions:
- fade between menu and gameplay
- short fade or slide into results
- no long blocking transitions

## 5. Session Flow
### First Session
- open app
- menu appears
- user taps play
- user learns by dragging and releasing
- user completes a round
- results screen appears

### Replay Session
- results screen offers restart
- game resets immediately
- previous high score remains available

## 6. Edge Cases
### Refresh Mid-Game
- safe reset to menu or saved default start state

### Pointer Cancel
- cancel current draw
- do not fire a phantom shot

### Resize / Orientation Change
- recalculate layout
- preserve state where possible
- keep target and HUD aligned

### Storage Unavailable
- game should still run
- high score can degrade gracefully

## 7. Flow Rules
- Never block the player with unnecessary screens.
- Never hide the core play action behind extra steps.
- Results should follow quickly after the round.
- Menu actions should always be obvious.

## 8. Flow Exit Criteria
The flow is correct when:
- a new player can move from launch to first shot in a few seconds,
- round completion is unambiguous,
- and restart is always easy to find.
