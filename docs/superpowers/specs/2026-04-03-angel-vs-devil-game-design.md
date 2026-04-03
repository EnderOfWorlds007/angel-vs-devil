# Angel vs Devil — Game Design Spec

## Overview

A 5-level educational multiplication campaign where a pixel-art angel powers up by solving math problems, then fights a unique boss in each world. Single-file HTML game (`angel-vs-devil.html`), no dependencies, works offline.

**Target player:** 8-9 year old kid
**Session length:** ~25 minutes for full campaign
**Core loop:** Solve multiplications -> power up angel -> fight boss -> advance

---

## Architecture

### Single Canvas + DOM Overlay (Approach A)

- **Game loop:** `requestAnimationFrame` drives `update(dt)` + `render()` every frame
- **State machine:** `gamePhase` string determines which update/render functions run
- **Canvas:** single `<canvas>` element, logical 800x500 aspect ratio (16:10), scaled to fit viewport
- **DOM overlay:** a `<div>` positioned over the canvas with identical scale transform, holds answer buttons, HUD elements, text
- **Sprite manager:** loads all base64 sprites at startup, extracts frames into offscreen canvases
- **Particle system:** array of particles with position/velocity/color/alpha/lifetime, updated and drawn each frame
- **Sound manager:** thin wrapper around Web Audio API, all 14 procedural sounds
- **Question generator:** produces `{a, b, answer, distractors[3]}` for the current level's table, tracks wrong answers

### State Machine

```
title -> level-intro -> training -> battle-transition -> battle -> victory/defeat
  victory -> level-intro (next level) OR campaign-complete
  defeat -> level-intro (retry same level, resets per-level state, keeps totalScore)
  campaign-complete -> title
```

### Responsive Scaling

- Maintain 16:10 aspect ratio (800x500 logical)
- On resize: compute largest rect that fits viewport at that ratio
- Set canvas CSS size + reposition DOM overlay to match
- Answer buttons use relative sizing, minimum 48px tall on any device
- On narrow viewports (phones landscape): answer buttons stack 2x2 instead of 1x4
- Works on tablet, laptop, and phone in landscape

---

## Campaign Structure

| Level | World | Table | Training Qs | Battle Qs | Battle Timer | Boss |
|---|---|---|---|---|---|---|
| 1 | Enchanted Forest | x2 | 6 | 5 | 10s | Shadow Wolf |
| 2 | Frozen Peaks | x3 | 9 | 6 | 9s | Frost Witch |
| 3 | Sunken Ruins | x4 | 12 | 7 | 8s | Kraken Lord |
| 4 | Volcanic Wastes | x5 | 14 | 8 | 8s | Devil Queen |
| 5 | The Dark Throne | x6 | 15 | 10 | 7s | Violet Avenger |

---

## Sprites

### Sprite Sources

| Key | Source | Used For | Format |
|---|---|---|---|
| `fallen2` | sprites_b64.json | Angel Stage 0 (dim) & Stage 1 (full) | 48x64, 3x4 grid, 16x16 frames |
| `angel` | sprites_b64.json | Angel Stage 2 -- Guardian | same |
| `archangel` | sprites_b64.json | Angel Stage 3 -- Archangel, Stage 4 -- Seraphim (+glow) | same |
| `werewolf` | OpenGameArt (Cabbit) | Level 1 boss -- Shadow Wolf | 48x64, SWEN |
| `witch` | OpenGameArt (Cabbit) | Level 2 boss -- Frost Witch (blue-tinted) | 48x64, SWEN |
| `kraken` | OpenGameArt (Stendhal) | Level 3 boss -- Kraken Lord | 96x128, SWEN |
| `devil` | sprites_b64.json | Level 4 boss -- Devil Queen | 48x64, SWEN |
| `violet` | sprites_b64.json | Level 5 boss -- Violet Avenger | 48x64, SWEN |

### Sprite Processing

1. Decode all base64 strings into `Image` objects at startup
2. Pre-extract all 12 frames (3 cols x 4 rows) into offscreen canvases per sprite
3. Store as `frames[spriteKey][direction][frameIndex]`
4. Frame extraction: `drawImage(img, col*16, row*16, 16, 16, ...)` for 48x64 sheets, `drawImage(img, col*32, row*32, 32, 32, ...)` for kraken (96x128)
5. Canvas uses `image-rendering: pixelated` for crisp upscaling
6. Render angel at ~128px tall (8x scale for 16px frames, 4x for kraken's 32px)

### Angel Evolution

| Stage | Name | Sprite | Visual Effect | Thresholds (L1/L2/L3/L4/L5) |
|---|---|---|---|---|
| 0 | Weakened Angel | `fallen2` | 50% opacity | Start |
| 1 | Awakened | `fallen2` | Full opacity | 1/2/2/3/3 correct |
| 2 | Guardian | `angel` | Blue/white robes | 3/4/5/6/7 correct |
| 3 | Archangel | `archangel` | Golden armor | 4/6/8/10/11 correct |
| 4 | Seraphim | `archangel` | +golden tint, aura, sparkle particles | 6/9/12/14/15 correct |

Evolution transition: 0.5s pause, flash + `sfxPowerUp()`, old sprite fades out, new fades in, stage name text appears, particle burst.

### Frost Witch Tinting

Apply blue/ice overlay to witch sprite using `source-atop` compositing (same technique as Seraphim golden glow but ice-blue). Add frost particle effects around her.

---

## Level Backgrounds

All backgrounds drawn procedurally on canvas, cached to offscreen canvas. Only particles animate per-frame. Redrawn on level/phase transitions.

### Level 1 -- Enchanted Forest

- **Sky:** dark blue-to-deep-green vertical gradient
- **Ground:** dark mossy green with grass tufts (triangle shapes)
- **Elements:** 2-3 gnarled tree silhouettes (bezier curves), stone ruin outline, glowing mushrooms (ellipses with radial glow)
- **Moonlight:** soft white radial gradient, light shafts (semi-transparent angled rectangles)
- **Particles:** fireflies -- small yellow-green dots drifting randomly, pulsing alpha

### Level 2 -- Frozen Peaks

- **Sky:** dark blue-to-purple gradient with aurora bands (sine-wave color strips in green/purple, slowly shifting)
- **Ground:** white/light-blue snow
- **Elements:** mountain peak silhouettes (triangles with snow caps), pine tree silhouettes
- **Particles:** snowfall -- white dots at slight angles, variable sizes for depth. Ice crystal sparkle flashes.

### Level 3 -- Sunken Ruins

- **Background:** deep blue-to-turquoise vertical gradient
- **Elements:** coral column silhouettes (edges), broken marble pillars, light shafts from above
- **Jellyfish:** small translucent shapes with trailing tentacles, drifting slowly
- **Particles:** bubbles rising -- circles with highlight dot, variable speed/size

### Level 4 -- Volcanic Wastes

- **Sky:** dark red-to-black gradient, crimson clouds
- **Ground:** cracked dark earth with lava rivers (orange-yellow glowing lines)
- **Elements:** distant volcano silhouettes, lava geysers (periodic column eruptions)
- **Particles:** embers floating upward -- orange/red dots with sway

### Level 5 -- The Dark Throne

- **Background:** deep purple-to-black void with warped stars
- **Elements:** obsidian castle silhouette, floating debris chunks, stained glass windows, floor runes pulsing
- **Particles:** dark energy motes -- purple/black dots in slow orbits. Periodic purple lightning.

### Battle Arena Variants

Same background system but shifted darker/more intense. Arena-specific framing elements added (trees for L1 clearing, ice crystals for L2, etc.).

---

## Training Phase

### Screen Layout

- **Top-left HUD:** level indicator, stage name, power bar (fills with correct answers), streak counter
- **Top-right HUD:** circular countdown timer (15s), current score
- **Center:** level background with angel character (center-left, south-facing, idle bobbing)
- **Bottom 25%:** question text prominently displayed + 4 answer buttons

### Question Generation

- Level N uses the x(N+1) table: Level 1 = x2, Level 2 = x3, ..., Level 5 = x6
- Generate all 10 possible questions (N x 1 through N x 10), shuffle
- No repeating the same multiplier twice in a row
- Training question count per level: 6, 9, 12, 14, 15

### Distractor Generation

- 3 wrong answers per question
- Strategy: adjacent products in the same table (+/-1 or +/-2 multiplier), plus one random offset (+/-1 to +/-5)
- No duplicates, no negatives, no zero, exactly one correct
- Shuffle all 4 options randomly across buttons

### On Correct Answer

- `sfxCorrect()` chime
- Screen flash (white overlay, fast fade)
- Power bar fills proportionally (100% / training question count)
- Streak counter increments
- Floating "+points!" text (speed-based)
- Check evolution thresholds: if stage changes, trigger evolution transition

### On Wrong Answer or Timeout

- `sfxWrong()` buzz
- Correct answer button highlights green for 1.5s (learning reinforcement)
- Wrong button flashes red
- Angel dims briefly (opacity dip for 0.5s)
- Streak resets to 0
- Question recorded in `wrongAnswers[]` for battle priority
- Gentle "Try again!" floating text

### On Streak of 3

- `sfxStreak()` arpeggio
- Sparkle burst particle effect around angel
- Bonus power added to bar
- "STREAK x3!" floating golden text

### Training Complete

- Sky darkens over 1 second
- Lightning flash
- "READY FOR BATTLE" text appears dramatically
- `sfxTransition()` whoosh
- 2-second pause, then transition to battle phase

---

## Boss Fight

### Battle Transition

- Screen fades to black
- Battle arena background fades in (darker variant)
- `sfxBattleStart()` horn + drum roll
- Boss entrance animation (unique per boss):
  - Shadow Wolf: lunges in with shadow trail
  - Frost Witch: floats in with ice crystal particles
  - Kraken Lord: tentacles rise from bottom, then head emerges
  - Devil Queen: fire erupts, she appears in flames
  - Violet Avenger: dark energy coalesces into her form
- Boss name + HP bar appear
- Angel on left (east-facing), boss on right (west-facing), both idle bobbing

### Screen Layout

- **Top-left:** Angel name + HP bar
- **Top-center:** circular countdown timer (more prominent than training, adds tension)
- **Top-right:** Boss name + HP bar
- **Center:** battle arena, angel left ~25%, boss right ~75%
- **Bottom 25%:** question + 4 answer buttons (same position as training for muscle memory)

### Question Priority Queue

1. Draw from `wrongAnswers[]` pool first (shuffled)
2. Once exhausted, fill remaining battle questions randomly from the level's table
3. If no wrong answers in training, all battle questions are random
4. Battle question counts: 5, 6, 7, 8, 10 per level

### Turn Flow

1. Question appears with timer
2. Player answers or timeout occurs
3. **Correct answer -> Angel attacks** (speed-based tier):
   - **DIVINE STRIKE** (<3s): 3x damage, angel lifts off, golden light column from above, particle explosion, heavy screen shake, `sfxDivineStrike()`
   - **HOLY SLASH** (<5s): 2x damage, golden crescent arc with sparkle trail, small screen shake, `sfxAttackHit()` with extra punch
   - **LIGHT BEAM** (<=timer): 1x damage, white beam shoots horizontally, boss flashes white, `sfxAttackHit()`
4. **Wrong answer or timeout -> Boss attacks** with unique animation, angel takes damage, `sfxDevilAttack()`
   - Correct answer highlights green for 1.5s before boss attacks (learning reinforcement)
5. 0.5s breathing pause (particles settle)
6. Next question

### Boss Attack Animations

| Boss | Attack | Duration | Visual |
|---|---|---|---|
| Shadow Wolf | Dark Bite | ~0.7s | Purple-black streak lunges across, shadowy jaw snap particles around angel |
| Frost Witch | Ice Shards | ~0.8s | 3-5 white-blue triangle shapes fly R-to-L with frost trail, angel flashes blue |
| Kraken Lord | Tentacle Slam | ~0.9s | Thick bezier curve tentacle sweeps across, water splash particles, screen shake |
| Devil Queen | Fire Blast | ~0.8s | Orange-red fireball arcs from boss to angel, ember particle explosion on impact |
| Violet Avenger | Dark Energy Bolt | ~1.0s | Screen dims, purple-black orb charges then fires as beam with spiraling particles |

### Angel Attack Animations

| Attack | Duration | Visual |
|---|---|---|
| LIGHT BEAM | ~0.6s | White-gold beam from angel's hand horizontal to boss. Glow trail particles. Boss flashes white 3x. |
| HOLY SLASH | ~0.8s | Golden crescent arc (bezier) sweeps angel-to-boss, trailing sparkles. Small screen shake. Boss knockback. |
| DIVINE STRIKE | ~1.2s | Angel lifts off, canvas scale 1.05x zoom, golden light column from above onto boss. Particle explosion, heavy screen shake, full screen flash. |

### Super Powers (every 3rd battle question)

Question glows golden, `sfxSuperPower()` shimmer plays. Before the question appears, three power icons are shown for the player to choose:

| Power | Icon | Effect | Visual |
|---|---|---|---|
| Wings of Fire | fire | Next attack does double damage (stacks with speed tier) | Angel's wings get orange-red flame particles. Next attack has fire mixed in. |
| Holy Lightning | lightning | Damages boss 1 hit AND heals angel 1 hit | Golden lightning bolts from top hit boss. Green healing pulse around angel. |
| Divine Shield | shield | Blocks next boss attack if player answers wrong | Golden translucent dome with hexagon pattern around angel. Shatters on hit. |

Player taps their choice, then the question appears. Power activates on resolution:
- **Wings of Fire:** activates only on correct answer (wasted if wrong -- no attack to boost)
- **Holy Lightning:** activates on correct answer (damage + heal happen alongside the angel's attack)
- **Divine Shield:** activates immediately and persists until the next wrong answer consumes it (or end of battle if unused)

### Damage & HP

| Level | Boss HP (normal hits) | Angel HP (boss hits) |
|---|---|---|
| 1 | 4 | 5 |
| 2 | 5 | 5 |
| 3 | 6 | 4 |
| 4 | 7 | 4 |
| 5 | 9 | 4 |

- Light Beam = 1 hit, Holy Slash = 2 hits, Divine Strike = 3 hits
- Boss attack = 1 hit of angel HP
- Wings of Fire doubles the hit value of the next attack
- Holy Lightning does 1 hit to boss + restores 1 angel HP
- Divine Shield absorbs 1 boss attack (if wrong answer)

### Hit Reactions

- **Flash:** target goes full white 3x over 0.3s (via `source-atop` compositing)
- **Knockback:** 10px displacement, spring back over 0.3s (ease-out)
- **Screen shake:** random +/- X/Y offset, 2px for slash, 6px for divine strike
- **Screen flash:** white overlay at 0.4 alpha, fades to 0 over 0.2s
- **Floating text:** attack name + damage, drifts upward and fades over 1s

### Win/Lose

- **WIN (boss HP <= 0):** boss shrinks, spins, dissolves into themed particles (1.5s), `sfxVictory()`, transition to victory screen
- **LOSE (angel HP <= 0):** angel kneels (south-facing frame), dims to 30% opacity, `sfxDefeat()`, encouraging message, retry option

---

## Scoring System

### Training Points (per correct answer)

| Speed | Points | Label |
|---|---|---|
| < 3s | 150 | "BLAZING!" |
| < 5s | 100 | "FAST!" |
| < 8s | 75 | "GOOD!" |
| <= 15s | 50 | "CORRECT" |

### Streak Multiplier (training only)

- 3 streak: 1.5x on that answer's points
- 5 streak: 2.0x
- 7+ streak: 2.5x

### Battle Points (per correct answer)

| Speed Tier | Points |
|---|---|
| DIVINE STRIKE (<3s) | 300 |
| HOLY SLASH (<5s) | 200 |
| LIGHT BEAM (<=timer) | 100 |

### Bonuses

- **Super power activation:** +50 points each time
- **No-damage bonus:** angel took zero boss hits -> +500
- **Perfect training:** zero wrong answers in training -> +300
- **Speed demon:** average answer time under 4s across whole level -> +200

### Level Scaling Multiplier

| Level | Multiplier |
|---|---|
| 1 | 1.0x |
| 2 | 1.2x |
| 3 | 1.4x |
| 4 | 1.6x |
| 5 | 2.0x |

All points earned in a level are multiplied by the level's multiplier. Approximate total score range: 8,000-15,000 for a decent run, up to ~25,000 for a perfect fast run.

---

## Screen Flow

### Title Screen

- Animated background (subtle particle effect)
- "ANGEL vs DEVIL" title with golden glow
- Angel sprite idle-animating below title
- "NEW CAMPAIGN" pulsing button
- Campaign progress dots (filled/empty circles for each level) if returning

### Level Intro

- World name fades in large: "LEVEL 3 -- SUNKEN RUINS"
- Background preview renders behind text
- Boss silhouette teased (dark outline)
- Flavor text: "The Kraken Lord awaits..."
- "BEGIN" button

### Victory Screen

- Angel ascends with particle trail
- World-specific celebration effects
- Stats: questions answered, accuracy %, fastest answer, best streak
- Score breakdown: base + speed + streak + completion bonuses x level multiplier
- "NEXT LEVEL" button

### Defeat Screen

- Somber but encouraging tone
- "You dealt X% damage to [Boss Name]!"
- Progress bar showing how close they got
- "The [Boss] is weakened. Try again!"
- "RETRY" button (score resets for that level only)

### Campaign Complete (after Level 5)

- Grand celebration -- golden particles everywhere
- All 5 boss silhouettes shown defeated in a row
- "YOU SAVED THE REALM!" text
- Final stats: total score, overall accuracy, total time
- Credits: sprite attribution
- "PLAY AGAIN" button

### Transitions

- `sfxTransition()` whoosh on every major transition
- Fade-to-black (0.3s) -> new screen fades in (0.3s)
- Level intro -> training: backdrop draws in as fade completes
- Training -> battle: dramatic darkening + lightning flash before fade

---

## Sound Effects

All 14 sounds procedural via Web Audio API (from `sound-effects-preview.html`):

### Training
- **Correct answer:** sine C5->G5 + triangle sparkle
- **Wrong answer:** square 220->140Hz descending buzz
- **Streak bonus:** C5-E5-G5-C6 arpeggio + shimmer
- **Power-up evolution:** sawtooth sweep + square fanfare

### Battle
- **Angel attack hit:** noise burst + triangle 880->440Hz ring
- **Divine strike:** whoosh + impact + angelic chord
- **Boss attack:** sawtooth 120->50Hz + noise crackle + lowpass
- **Super power activated:** sine sweep + triangle sparkles + chord

### UI & Transitions
- **Timer tick:** sine 880Hz, 80ms
- **Timer urgent (<3s):** square 1200Hz double pulse
- **Phase transition:** sawtooth sweep + bandpass filter
- **Battle start:** sawtooth horn + square melody + noise drums
- **Victory fanfare:** square melody C-E-G-C + triangle bass
- **Defeat:** triangle C5-B4-A4-E4 descending

---

## State Management

All state in JavaScript variables (no localStorage).

```
// Campaign state
currentLevel: 1-5
totalScore: number
levelsCompleted: [boolean x 5]

// Per-level state (resets each level)
gamePhase: "title" | "level-intro" | "training" | "battle-transition" | "battle" | "victory" | "defeat" | "campaign-complete"
angelStage: 0-4 (evolution stage)
powerBar: 0-100 (percentage)
streakCount: number
angelHP: number
bossHP: number
levelScore: number
questionsAnswered: number
correctAnswers: number
wrongAnswers: [{a, b}]
battleQuestionsRemaining: number
activeSuperPower: null | "wings-of-fire" | "holy-lightning" | "divine-shield"
hasShield: boolean
```

---

## Attribution

Sprites by Jordan Irwin (AntumDeluge), Svetlana Kushnariova (Cabbit), Stephen Challener (Redshrike), William Thompson, Lanea Zimmerman, Diligent Dodo (soniccuz). Licensed under CC-BY 3.0 / CC-BY 4.0 / OGA-BY 3.0 / CC-BY-SA 3.0. Attribution displayed in campaign complete credits screen.

### Sprite Sources

- Werewolf (Shadow Wolf): https://opengameart.org/content/werewolf-cabbit
- Witch on Broomstick (Frost Witch): https://opengameart.org/content/witch-on-broomstick
- Kraken (Kraken Lord): https://opengameart.org/content/kraken
- Angel/Devil sprites: sprites_b64.json (bundled)
