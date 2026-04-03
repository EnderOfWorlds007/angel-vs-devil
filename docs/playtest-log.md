# Playtest Log — Angel vs Devil

## Date: 2026-04-03

## Playtest Method

Automated playthrough using Puppeteer controlling Chrome. Played all 5 levels, tested victory, defeat+retry, and campaign complete flows.

## Console Errors

None

## Critical Bugs Found and Fixed

### 1. Training buttons broken after first battle (FIXED)

- **Symptom**: L2+ training questions appeared but clicking answers had no effect
- **Root cause**: `btn.dataset.mode = "battle"` set during L1 battle was never cleared when L2 training started. Clicks routed to `handleBattleAnswer()` instead of `handleAnswer()`
- **Fix**: Added `dataset.mode = ""` reset loop in `startTraining()`

### 2. Boss sprites not used for levels 1-3 (FIXED)

- **Symptom**: Shadow Wolf, Frost Witch, Kraken rendered as tiny programmatic shapes
- **Root cause**: `drawBossOnCanvas()` used programmatic drawing for levels 0-2 but sprites for 3-4
- **Fix**: All 5 bosses now render from sprite sheets with themed tint effects

## Full Campaign Playthrough

- **[LOAD]** Phase:title, zero JS errors
- **[L1 TRAINING]** x2 table, 6 Qs. Evolution 0->1->2->3. Score: 825. 1 wrong answer tracked
- **[L1 BATTLE]** Angel HP:5, Boss HP:4. Won in 2 turns (Divine Strike each). Total: 2,125
- **[L2 TRAINING]** x3 table, 9 Qs. Evolution 0->1->2->3->4 (Seraphim!). Score: 2,475
- **[L2 BATTLE]** Won. Running total: 7,015
- **[L3 DEFEAT TEST]** Intentionally answered wrong. HP: 5->3->2->1->0. Defeat screen worked
- **[L3 RETRY]** Retry returned to level-intro correctly
- **[L3 WIN]** Replayed correctly. Total: 14,295
- **[L4 WIN]** x5 table complete. Total: 24,295
- **[L5 WIN]** x6 table complete. Total: 37,545
- **[CAMPAIGN COMPLETE]** "YOU SAVED THE REALM!" with score and credits

## Visual Verification

| Screen | Status | Notes |
|---|---|---|
| Title screen | Good | Golden title, angel sprite, forest bg, mushrooms, moon |
| Level intro | Good | World name, boss preview, BEGIN button |
| Training HUD | Good | Power bar, timer, streak, level/score |
| Angel evolution | Good | All 5 stages render correctly |
| Seraphim glow | Good | Golden aura and sparkles visible |
| Frost Witch tint | Good | Blue-ice overlay on sprite |
| Shadow Wolf | Good | Werewolf sprite with purple aura |
| Battle layout | Good | Angel vs boss, HP bars, timer, questions |
| Victory screen | Good | Stats, score breakdown, NEXT LEVEL |
| Defeat screen | Good | Damage %, RETRY button works |
| Campaign complete | Good | Total score, credits, PLAY AGAIN |
