# Angel vs Devil — Multiplayer Mode Design Spec

## Overview

Split-screen PvP multiplayer on the same device. One kid plays the Angel, the other plays the Devil (boss). Both shoot at each other and answer multiplication questions to unlock super powers. Best of 3 rounds, roles swap each round.

## File Structure

```
angel-vs-devil.html    — Campaign mode (existing, add MULTIPLAYER button linking to multiplayer.html)
multiplayer.html        — Multiplayer mode (new)
assets.js               — Shared code extracted from campaign (sprites, sounds, particles, effects, backgrounds, drawing, questions)
sprites_b64.json        — Sprite data (existing)
```

### assets.js Contains
- Sprite loading and frame extraction (all sprite keys including dark_angel, fallen3)
- All 14 sound effect functions (sfxCorrect, sfxWrong, sfxStreak, sfxPowerUp, sfxAttackHit, sfxDivineStrike, sfxDevilAttack, sfxSuperPower, sfxTimerTick, sfxTimerUrgent, sfxTransition, sfxBattleStart, sfxVictory, sfxDefeat)
- Particle system (particles array, addParticle, spawnParticleBurst, updateParticles, renderParticles)
- activeEffects system + ALL effect renderers (lightBeam, holySlash, divineStrike, lightBolt, holyArrow, sacredLance, divineBarrage, fireWingIgnite, fireSelectTint, fireAura, fireBeamOverlay, fireImpactExplosion, holyLightningBolt, holyHealAura, shieldDomeMaterialise, shieldHexPattern, shieldShatter, shieldShatterRing, shadowJaw, shadowSwoosh, frostShard, frostBurst, krakenTentacle, devilFireball, fireExplosion, violetChargeOrb, violetBeam, violetImplosion, violetCracks, screenTint, screenDim, lightningSelectFlash, shockwave, starBurst)
- Background drawing functions (all 5 levels + ambient particles)
- Angel drawing (drawAngel with all stages, mode-aware sprite selection, aura/glow/flame effects, seraphim sparkles)
- Boss drawing (drawBossOnCanvas with all 5 bosses + tint effects)
- Attack spawn functions (spawnFeebleSpark, spawnLightBolt, spawnHolyArrow, spawnSacredLance, spawnDivineBarrage, spawnLightBeam, spawnHolySlash, spawnDivineStrike, spawnBossAttack + all 5 boss attack variants)
- Question generator (generateQuestionQueue, generateDistractors, generateBattleQuestions)
- Number word lookup for voice recognition (EN + FR, 1-60)
- Screen effects (triggerShake, triggerFlash)
- Utility (shuffleArray)
- Sprite data embedded (read from sprites_b64.json at build time — inline the base64 strings)

### angel-vs-devil.html Changes
- Extract shared code into assets.js, add `<script src="assets.js">`
- Add "MULTIPLAYER" button on title screen that navigates to `multiplayer.html`
- Campaign-specific code stays in the HTML (state machine, training phase, battle phase, campaign flow, UI event listeners)

### multiplayer.html Contains
- HTML/CSS for split-screen layout, lobby, and match UI
- `<script src="assets.js">`
- Multiplayer game loop, state, input routing, per-side rendering
- Voice recognition module
- "CAMPAIGN" button linking back to angel-vs-devil.html

---

## Game Flow

### 1. Lobby Screen
- "ANGEL vs DEVIL — MULTIPLAYER" title
- Level select: 5 world buttons (Enchanted Forest through Dark Throne). Selecting a level shows its background as preview.
- Difficulty: Easy / Normal / Hard toggle (same MODE_CONFIG as campaign)
- Language: EN / FR toggle (for voice recognition)
- Keyboard layout hint showing the split
- "START MATCH" button

### 2. Pre-Round Screen (~3 seconds)
- Shows "ROUND 1 of 3"
- "Player 1: ANGEL (left) | Player 2: DEVIL (right)"
- Roles swap each round
- Brief countdown: 3... 2... 1... FIGHT!

### 3. Training Phase (split screen, ~30 seconds max)
- Left half: angel player's training (questions, power bar, evolution)
- Right half: devil player's training (questions, power bar, boss evolution)
- Same multiplication table (determined by level selection)
- Independent timers (15s normal, same mode adjustments as campaign)
- Both players use the SAME evolution thresholds as campaign (stages 0-4). Devil evolves through boss stages using the same correct-answer counts. 6 training questions for all levels in multiplayer (keeps it quick).
- Power bar fills independently per side
- Training ends for a player when their bar hits 100% OR all 6 questions answered
- When a player finishes first: their side shows "READY!" with their evolved character posing. They watch the other side finish.
- First player to finish gets +1 shield charge in battle (not HP, to avoid snowballing)
- When both done: "READY FOR BATTLE!" transition
- If 30 seconds pass, both sides are force-completed at whatever stage they reached

### 4. Battle Phase (split screen)

**Screen layout:**
```
|  LEFT 25%  |     CENTER 50%     |  RIGHT 25%  |
| Angel HUD  |                    | Devil HUD   |
| HP bar     |    Angel ← → Boss  | HP bar      |
| Power bar  |    (battlefield)   | Power bar   |
|            |                    |             |
| Question   |                    | Question    |
| [1][2][3][4]                    | [9][0][-][=]|
```

**HP (PvP balanced — both players equal):**
- Both players: 15 HP (normal mode)
- Easy: 18 HP each
- Hard: 12 HP each
- First-to-finish training bonus: +1 shield charge (not raw HP, to avoid snowballing)

**Shooting:**
- Angel player shoots with left-hand keys: Q W E R T A S D F G Z X C V B
- Devil player shoots with right-hand keys: Y U I O P [ ] H J K L ; ' N M , . /
- Each keypress fires that player's attack (rate limited to max 2/sec)
- Space, mouse clicks, Tab, Shift, Ctrl, Cmd, Escape, Fn keys do NOT fire
- Attack visual scales with evolution stage (same 5 tiers: feeble spark → divine barrage for angel, dark equivalents for devil)
- Stages 3-4 do 2 damage per shot

**Questions:**
- Each player gets independent questions on a ~6 second cycle
- Questions appear in their HUD zone (left or right)
- Answer keys: Angel uses 1/2/3/4, Devil uses 9/0/-/=
- Keys shown on the answer buttons
- Same keys serve double duty: answers when question showing, powers when power selection showing, ignored otherwise (shooting uses letter keys)
- Wrong answer / timeout: opponent gets a free 2-damage hit

**Mouse/Touch (wild card):**
- Clicking/tapping an answer button on EITHER side registers for that side
- Hit-test by X coordinate to determine which side
- This means either player can grab the mouse and answer for either side (including sabotage!)

**Voice (wild card):**
- Web Speech API runs continuously during battle
- Matches spoken numbers against both EN and FR word tables
- When a match is found, checks all visible answer buttons on both sides
- If the number matches the CORRECT answer on a side, it triggers for that side
- If it matches correct answers on BOTH sides simultaneously, the side whose question appeared first gets it
- Voice is intentionally chaotic — kids shouting answers at each other IS the fun. This is a feature, not a bug.
- Language toggle in lobby determines primary recognition language, but both word tables are always checked

**Super Powers:**
- Correct answer → power selection overlay appears ON THAT PLAYER'S SIDE ONLY
- Angel powers (keys 1-4): Wings of Fire, Holy Lightning, Divine Shield, Healing Potion
- Devil powers (keys 9/0/-/=): Hellfire Blast, Dark Lightning, Shadow Shield, Blood Potion
- Same mechanics as campaign (fire: 5 dmg, lightning: 3 dmg + 2 heal, shield: block 2, heal: +4 HP)

**Devil power visuals:**

| Devil Power | Key | Visual |
|---|---|---|
| Hellfire Blast | 9 | Dark red/black fireball with skull smoke, opponent's side tints red |
| Dark Lightning | 0 | Purple lightning bolts rising from below, dark green heal mist |
| Shadow Shield | - | Dark purple dome with skull facets, shatters into black shards |
| Blood Potion | = | Red swirling particles converging inward, crimson pulse |

**Devil shooting attack visuals (per stage):**

| Stage | Name | Visual |
|---|---|---|
| 0 | Weakened | Feeble dark spark (grey-purple, dim) |
| 1 | Awakened | Dark bolt (purple beam, short) |
| 2 | Empowered | Shadow arrow (dark teal/purple streak with trail) |
| 3 | Enraged | Dark lance (red-black beam with afterglow) |
| 4 | Unleashed | Inferno barrage (wide dark beam with fire spirals + lightning) |

**Battle end:**
- Win when opponent's HP reaches 0
- Winner's character does victory pose, loser does defeat animation
- Only affects the battle zone (center), HUDs stay visible
- **Simultaneous kill (both reach 0 on same frame):** draw — neither player scores a point. Round replays with same roles.
- **Best of 3 tiebreak:** if it's 1-1 after round 3 (due to a draw), play a sudden-death round 4 (both start at 8 HP, same roles as round 3)

### 5. Round End Screen
- "PLAYER 1 WINS!" or "PLAYER 2 WINS!"
- Score: "Round 1: Angel wins | Round 2: ? | Round 3: ?"
- Roles swap announcement: "Swapping roles! Player 1 is now the DEVIL"
- "NEXT ROUND" button (or auto-advance after 5 seconds)

### 6. Match End Screen
- "PLAYER [1/2] IS THE CHAMPION!"
- Final score (e.g., "2 - 1")
- Stats per player: shots fired, accuracy, powers used
- "REMATCH" button (same settings)
- "BACK TO LOBBY" button
- "CAMPAIGN" button (link to angel-vs-devil.html)

---

## Voice Recognition

```javascript
const NUMBER_WORDS = {
  // English
  "one":1,"two":2,"three":3,"four":4,"five":5,"six":6,"seven":7,"eight":8,
  "nine":9,"ten":10,"eleven":11,"twelve":12,"thirteen":13,"fourteen":14,
  "fifteen":15,"sixteen":16,"seventeen":17,"eighteen":18,"nineteen":19,
  "twenty":20,"twenty one":21,"twenty two":22,"twenty three":23,"twenty four":24,
  "twenty five":25,"twenty six":26,"twenty seven":27,"twenty eight":28,
  "twenty nine":29,"thirty":30,"thirty one":31,"thirty two":32,"thirty three":33,
  "thirty four":34,"thirty five":35,"thirty six":36,"forty":40,"forty two":42,
  "forty eight":48,"fifty":50,"fifty four":54,"sixty":60,
  // French
  "un":1,"deux":2,"trois":3,"quatre":4,"cinq":5,"six":6,"sept":7,"huit":8,
  "neuf":9,"dix":10,"onze":11,"douze":12,"treize":13,"quatorze":14,
  "quinze":15,"seize":16,"dix sept":17,"dix-sept":17,"dix huit":18,"dix-huit":18,
  "dix neuf":19,"dix-neuf":19,"vingt":20,"vingt et un":21,"vingt-et-un":21,
  "vingt deux":22,"vingt-deux":22,"vingt trois":23,"vingt-trois":23,
  "vingt quatre":24,"vingt-quatre":24,"vingt cinq":25,"vingt-cinq":25,
  "vingt six":26,"vingt-six":26,"vingt sept":27,"vingt-sept":27,
  "vingt huit":28,"vingt-huit":28,"vingt neuf":29,"vingt-neuf":29,
  "trente":30,"trente six":36,"trente-six":36,"quarante":40,"quarante deux":42,
  "quarante-deux":42,"quarante huit":48,"quarante-huit":48,"cinquante":50,
  "cinquante quatre":54,"cinquante-quatre":54,"soixante":60,
};
```

- Run `SpeechRecognition` with `continuous: true`, `interimResults: true`
- Primary `lang` set from lobby toggle (en-US or fr-FR)
- On each result, normalize transcript (lowercase, trim), look up in NUMBER_WORDS table
- Also try `parseInt()` on the transcript (speech recognition sometimes returns digits)
- If a number matches any visible answer button on either side, trigger that answer

---

## Input Routing Summary

```
KEY PRESSED
  ├── 1,2,3,4 → Left player action (answer OR power, context-dependent)
  ├── 9,0,-,= → Right player action (answer OR power, context-dependent)
  ├── Q,W,E,R,T,A,S,D,F,G,Z,X,C,V,B → Left player SHOOT
  ├── Y,U,I,O,P,[,],H,J,K,L,;,',N,M,comma,period,/ → Right player SHOOT
  └── Space, Tab, Shift, Ctrl, Cmd, Esc, Fn → IGNORED

MOUSE/TOUCH
  ├── Click on answer/power button → That side's action
  └── Other clicks → IGNORED (no shooting by mouse)

VOICE
  └── Spoken number → Match against visible answers on both sides, first match wins
```

---

## Rendering

- Single canvas, full viewport width
- Three clip zones: left 25%, center 50%, right 25%
- Battlefield renders in center zone (both characters, all attacks/particles/effects)
- HUD renders in side zones (HP bar, power bar, stage name, question, answer buttons)
- Per-side screen effects: shake and flash only affect that side's zone + center zone
- Particles and effects that cross zone boundaries are fine (adds to the chaos)

---

## Balance (PvP)

- Both players have equal HP: 15 (normal), 18 (easy), 12 (hard)
- Both evolve through same stages with same thresholds
- Both have same shot rate limit (2/sec) and same damage (1 per shot, 2 at stage 3-4)
- Both get questions at same frequency (~6 second cycle)
- Super powers do same damage regardless of side (5/3/block 2/heal 4)
- Wrong answer penalty is same (opponent gets free 2 damage hit)
- No boss fatigue in PvP (both players are human — no adaptive AI needed)
- First-to-finish training bonus: +2 HP (rewards fast math)

---

## Attribution

Same as campaign: Jordan Irwin (AntumDeluge), Svetlana Kushnariova (Cabbit), Stephen Challener (Redshrike), William Thompson, Lanea Zimmerman, Diligent Dodo (soniccuz). CC-BY 3.0/4.0, OGA-BY 3.0, CC-BY-SA 3.0.
