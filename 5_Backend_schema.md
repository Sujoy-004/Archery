# Backend Schema

## MVP Decision
No backend is required for the MVP.

## Reason
The product is:
- single-player,
- offline-first,
- and local-persistence only.

## What is stored locally
- high score
- optional settings in later versions

## Future Backend Only If Scope Expands
If the product later adds accounts, cloud saves, or leaderboards, a backend can be introduced with a simple schema.

### Future Entities
#### User
- id
- displayName
- createdAt
- lastActiveAt

#### ScoreEntry
- id
- userId
- mode
- score
- shotsFired
- createdAt

#### Match
- id
- userId
- mode
- score
- duration
- deviceType
- createdAt

#### LeaderboardEntry
- id
- userId
- mode
- bestScore
- updatedAt

## Future API Needs
- save score
- fetch best score
- fetch leaderboard
- sync user settings

## Recommendation
Do not build backend infrastructure until the MVP proves retention and replay value.
