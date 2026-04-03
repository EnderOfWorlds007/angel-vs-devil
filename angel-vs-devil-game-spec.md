# Angel vs Devil — Math Battle Campaign

## Spec for Claude Code

> A 5-level educational multiplication campaign where a pixel-art angel powers up by solving math problems, then fights a unique boss in each world. Difficulty escalates across levels.

---

## Overview

**Target player:** 8–9 year old kid
**Platform:** Single-file HTML game (runs in any browser, no dependencies)
**Art style:** Real pixel-art sprites (48×64, embedded as base64) with procedural backgrounds
**Session length:** ~25 minutes for the full campaign (~3–6 min per level)
**Core loop:** Solve multiplications → power up angel → fight boss → advance to next world

---

## Campaign Structure

The game is a 5-level campaign. Each level has a unique world backdrop, a boss with its own personality, and escalating difficulty. The angel's evolution resets at the start of each level (so the power-up loop always feels fresh), but the player's total score carries across levels.

### Level Overview

Each level focuses on a **single multiplication table**, building mastery one table at a time.

| Level | World | Table | Training Qs | Battle Qs | Boss |
|---|---|---|---|---|---|
| 1 | Enchanted Forest | ×2 | 6 | 5 | Shadow Wolf |
| 2 | Frozen Peaks | ×3 | 9 | 6 | Frost Witch |
| 3 | Sunken Ruins | ×4 | 12 | 7 | Kraken Lord |
| 4 | Volcanic Wastes | ×5 | 14 | 8 | Devil Queen |
| 5 | The Dark Throne | ×6 | 15 | 10 | Violet Avenger |

---

## World Descriptions & Backdrops

### Level 1 — Enchanted Forest
**Backdrop:** Deep green canopy with shafts of moonlight. Fireflies drift across the screen. Mossy ground, glowing mushrooms, a stone ruin in the background. Peaceful but mysterious.
**Training mood:** Calm, inviting — this is the tutorial level.
**Battle arena:** A moonlit clearing. Gnarled trees frame the edges. Mist curls along the ground.
**Boss — Shadow Wolf:** A dark wolf spirit with glowing red eyes and smoky purple aura. Lunges and howls when attacking. Cartoonish, not scary.
**Music vibe:** Gentle chiptune with nature sounds.

### Level 2 — Frozen Peaks
**Backdrop:** Snow-covered mountain peaks under an aurora borealis sky (green and purple lights). Pine trees dusted with snow. Gentle snowfall particle effect.
**Training mood:** Brisk, energizing. Breath-mist particle on the angel.
**Battle arena:** An icy plateau overlooking a frozen lake. Jagged ice crystals frame the scene. Northern lights dance above.
**Boss — Frost Witch:** A pale blue sorceress hovering in the air, ice crown, flowing frost robes. Attacks with ice shards. Cackles when hitting the angel.
**Music vibe:** Crisp, higher-pitched chiptune. Crystalline sound effects.

### Level 3 — Sunken Ruins
**Backdrop:** Underwater ancient temple. Coral columns, glowing jellyfish drifting by, shafts of light from the surface above. Bubble particles rising. Turquoise and deep blue color palette.
**Training mood:** Mysterious, wonderous. Slow bubble particles.
**Battle arena:** The temple's inner sanctum. Broken marble pillars, a glowing altar, dark water in the background. Bioluminescent creatures watch from the shadows.
**Boss — Kraken Lord:** A massive tentacled demon head rising from the deep. Purple and teal coloring. Slams tentacles when attacking. Deep rumbling sound effects.
**Music vibe:** Echoing, watery chiptune with reverb.

### Level 4 — Volcanic Wastes
**Backdrop:** Cracked earth with rivers of lava. Ash clouds in a dark red sky. Distant volcanoes erupting. Ember particles floating up. This is the original battle backdrop from our mockups.
**Training mood:** Intense, urgent. Orange glow on the angel.
**Battle arena:** A rock platform suspended over a lava lake. Lava geysers erupt periodically. The sky is deep crimson.
**Boss — Devil Queen:** Our existing devil queen sprite. Dark wings, crown of fire, arcane staff. The classic villain. Attacks with fire blasts and dark magic.
**Music vibe:** Driving, aggressive chiptune. Heavy bass.

### Level 5 — The Dark Throne
**Backdrop:** A floating obsidian castle in a void of swirling dark energy. Purple lightning crackles. Stars and galaxies visible in the distance but warped and corrupted. Eerie floating debris.
**Training mood:** Epic, foreboding. Dark particle effects, purple lightning flashes.
**Battle arena:** The throne room itself. Enormous dark throne behind the boss. Stained glass windows showing corrupted scenes. Magical runes pulse on the floor.
**Boss — Violet Avenger:** Our existing violet avenger sprite. The final boss — a dark sorceress at the peak of her power. Multiple attack types, faster attack speed. The ultimate challenge.
**Music vibe:** Epic final-battle chiptune. Dramatic key changes.

---

## Game Flow

### 1. Title Screen
- Title "Angel vs Devil" with animated angel character
- "NEW CAMPAIGN" button (glowing/pulsing)
- Campaign progress indicator (shows which levels have been completed in this session)
- Total score display

### 2. Level Intro Screen
- World name fades in dramatically (e.g., "LEVEL 1 — ENCHANTED FOREST")
- Brief backdrop preview with the boss silhouette teased
- "The Shadow Wolf awaits..." flavor text
- "BEGIN" button

### 3. Phase 1 — Power-Up Training

The angel starts as the Weakened Angel each level. The screen shows:

- **The world backdrop** — unique per level
- **The angel character** (center-left) with visible evolution
- **A power bar** that fills as questions are answered correctly
- **A multiplication question** displayed prominently (e.g., "7 × 8 = ?")
- **4 answer buttons** (one correct, three plausible wrong answers)
- **A streak counter** showing consecutive correct answers
- **A timer** (15 seconds per question — generous, not punishing)
- **Level indicator** (top corner)

#### Angel Evolution (resets each level)

The evolution thresholds scale with the level's training question count:

| Stage | Name | Visual Change | Threshold (fraction of level's training Qs) |
|---|---|---|---|
| 0 | Weakened Angel | Faded, dim sprite (cream robes, grey wings) | Start |
| 1 | Awakened | Same sprite at full brightness | ~20% |
| 2 | Guardian | Blue/white robes, white wings, golden halo | ~45% |
| 3 | Archangel | Golden armor, red cape, sword | ~70% |
| 4 | Seraphim | Same + blazing golden aura, particles, glow | 100% |

Concrete thresholds per level:

| Level | Training Qs | Stage 1 at | Stage 2 at | Stage 3 at | Stage 4 at |
|---|---|---|---|---|---|
| 1 | 6 | 1 | 3 | 4 | 6 |
| 2 | 9 | 2 | 4 | 6 | 9 |
| 3 | 12 | 2 | 5 | 8 | 12 |
| 4 | 14 | 3 | 6 | 10 | 14 |
| 5 | 15 | 3 | 7 | 11 | 15 |

#### Power-Up Mechanics

- Each correct answer fills the power bar and triggers a chime + screen flash
- Wrong answers: gentle "try again" — the angel dims slightly but doesn't lose a level
- **Streak bonus:** 3 correct in a row = bonus power burst + sparkle animation
- When the power bar is full, the sky darkens, lightning flashes, "READY FOR BATTLE" appears
- Dramatic transition to Phase 2

#### Question Difficulty Per Level

Each level focuses on a **single multiplication table**. Questions are of the form `N × ?` where N is the level's table number and `?` ranges from 1–10.

- **Level 1 (×2):** 2×1 through 2×10. The gentlest intro — doubles are intuitive.
- **Level 2 (×3):** 3×1 through 3×10. Introduces the first tricky table.
- **Level 3 (×4):** 4×1 through 4×10. Builds on doubling skills from level 1.
- **Level 4 (×5):** 5×1 through 5×10. The satisfying fives — patterns with 0 and 5.
- **Level 5 (×6):** 6×1 through 6×10. The hardest table — requires real mastery.

Within each level, questions are shuffled randomly but avoid repeating the same multiplier twice in a row. The second factor ranges from 1–10.

Wrong answer distractors should be "plausible" — close to the real answer (±1–5, or products from adjacent multiplications in the same table). Ensure exactly one option is correct.

### 4. Phase 2 — Boss Fight

The screen transitions to the level's battle arena. The boss appears on the right side.

- **Angel** (left side) with accumulated power visualized
- **Boss** (right side) with a health bar and name
- **Angel's health bar** (starts full)
- **Battle multiplication questions** — drawn from the level's table, **prioritising questions the player got wrong during training**

#### Wrong-Answer Priority Queue

During training, the game tracks every question the player answered incorrectly (stored as a `wrongAnswers` array of `{a, b}` pairs). When generating battle questions:

1. **First**, draw from the `wrongAnswers` pool (shuffled). This gives the kid a second chance at the ones they struggled with.
2. **Once the wrong-answer pool is exhausted**, fill remaining battle questions randomly from the level's table (avoiding repeats where possible).
3. If the player got everything right in training — great! Battle questions are fully random from the table.

This means a kid who struggles with e.g. 4×7 and 4×9 in training will face those exact questions again during the boss fight, reinforcing learning through repetition.

#### Battle Mechanics

Turn-based with a timer twist:

1. A multiplication question appears (8-second timer, faster than training)
2. **Correct answer → Angel attacks!**
   - Attack power scales with answer speed:
     - Under 3 seconds: **DIVINE STRIKE** — 3x damage, big dramatic animation
     - Under 5 seconds: **HOLY SLASH** — 2x damage
     - Under 8 seconds: **LIGHT BEAM** — 1x damage
3. **Wrong answer → Boss attacks!**
   - The boss does an attack, angel loses some health
   - A "shield" can absorb one hit if the player had a streak bonus from training

4. **Super Power moments** (every 3rd question in battle):
   - The question glows golden
   - Correct answer triggers a **SUPER POWER**:
     - "Wings of Fire" — next attack does double damage
     - "Holy Lightning" — damages boss AND heals angel slightly
     - "Divine Shield" — blocks the next boss attack even if wrong

#### Combat Visuals — How Attacks Look

Combat is **spell-based** — no physical contact. The angel and boss stand on opposite sides of the screen and cast magic at each other. All attacks are canvas particle effects, glows, and screen shakes — no complex sprite animation needed.

##### Angel Attacks (on correct answer)

| Attack | Trigger | Visual Effect |
|---|---|---|
| **LIGHT BEAM** | Correct, slow (5–8s) | Angel raises hand → a white beam of light shoots horizontally across the screen and hits the boss. Simple glow trail. Boss flashes white on impact. |
| **HOLY SLASH** | Correct, medium (3–5s) | Angel swings arm → a golden crescent arc sweeps from the angel across to the boss, trailing sparkles. Boss knocked back slightly + flash. Screen shake (small). |
| **DIVINE STRIKE** | Correct, fast (<3s) | Angel lifts off the ground, wings spread wide → camera zooms slightly → a massive column of golden light slams down on the boss from above. Particle explosion, screen flash, heavy screen shake. The most satisfying attack. |

All angel attacks use the **same color palette** — white, gold, and soft blue — to keep the holy/divine theme consistent.

##### Super Power Visuals

| Power | Visual Effect |
|---|---|
| **Wings of Fire** | Angel's wings ignite with orange-red flames. Flame particles trail from the wings for the duration. Next attack has fire mixed into its particles. |
| **Holy Lightning** | Golden lightning bolts crack down from the top of the screen hitting the boss. Simultaneously, a soft green healing glow pulses around the angel. Electric particles scatter. |
| **Divine Shield** | A translucent golden dome/bubble appears around the angel with hexagonal facets. Glows gently. When a boss attack hits it, it shatters into golden fragments with a satisfying sound. |

##### Boss Attacks (on wrong answer)

Each boss has a **unique attack animation** matching their world theme:

| Boss | Attack Name | Visual Effect |
|---|---|---|
| **Shadow Wolf** | Dark Bite | Wolf lunges forward as a dark shadow streak → shadowy jaw snap effect around the angel. Purple-black particles. |
| **Frost Witch** | Ice Shards | Witch raises staff → 3–5 jagged ice crystals fly from right to left, trailing frost particles. Angel flashes blue on impact. |
| **Kraken Lord** | Tentacle Slam | A huge dark tentacle sweeps across the screen from the right side. Water splash particles on impact. Screen shake. |
| **Devil Queen** | Fire Blast | Queen hurls a swirling fireball (orange-red with dark core). Travels in an arc. Explosion of ember particles on impact. |
| **Violet Avenger** | Dark Energy Bolt | Sorceress charges a purple-black energy orb → fires it as a beam with spiraling dark particles. The most intimidating attack — screen dims slightly during charge-up. |

##### General Combat Feel

- **Idle animation:** Both characters bob gently (1–2px vertical oscillation) while waiting for the player to answer.
- **Hit reaction:** When hit, characters flash white 3 times rapidly and knock back ~10px before returning to position.
- **Defeat animation:** Boss shrinks, spins, and dissolves into particles matching their theme color. Angel (if defeated) kneels and dims.
- **Between turns:** Brief 0.5s pause with no question shown — just the aftermath particles settling — before the next question appears. This gives each exchange breathing room.

#### Boss Scaling

| Level | Boss HP (in normal hits) | Angel HP (in boss hits) | Boss attack damage |
|---|---|---|---|
| 1 | 4 | 5 | Low |
| 2 | 5 | 5 | Low-Medium |
| 3 | 6 | 4 | Medium |
| 4 | 7 | 4 | Medium-High |
| 5 | 9 | 4 | High |

Balance target: ~85% win rate on level 1, ~70% on level 5 (for a kid who knows tables reasonably well).

#### Win/Lose Conditions

- **WIN:** Boss health reaches 0 → victory animation, score bonus, advance to next level
- **LOSE:** Angel health reaches 0 → encouraging message with progress shown ("You dealt 80% damage! Almost there!"), option to retry the level

### 5. Level Complete Screen

- Victory animation (angel ascends, fireworks, world-specific effects)
- Stats: questions answered, accuracy %, fastest answer, streak record
- Score breakdown: base score + speed bonuses + streak bonuses + no-damage bonus
- "NEXT LEVEL" button (or "CAMPAIGN COMPLETE" on level 5)

### 6. Campaign Complete Screen (after Level 5)

- Grand victory celebration — all 5 bosses shown defeated
- Final stats across all levels
- Total score
- "You saved the realm!" message
- "PLAY AGAIN" button to restart the campaign

---

## Sprite Assets

All sprites are from OpenGameArt.org (CC-BY 3.0/4.0 licensed, 48×64 pixels). They are embedded as base64 data URIs in the HTML file.

### Angel Sprites (5 evolution stages)
- **Stage 0 — Weakened Angel:** `angel-f-002` rendered at reduced opacity
- **Stage 1 — Awakened:** `angel-f-002` at full opacity
- **Stage 2 — Guardian:** `angel-f-001` (blue/white robes, white wings)
- **Stage 3 — Archangel:** `angel-m-001` (golden armor, red cape)
- **Stage 4 — Seraphim:** `angel-m-001` with golden tint overlay, aura, and particle effects

### Boss Sprites
- **Level 4 — Devil Queen:** `devil_queen` sprite from devil_queen_et_al pack
- **Level 5 — Violet Avenger:** `violet_avenger` sprite from same pack
- **Levels 1–3:** Bosses (Shadow Wolf, Frost Witch, Kraken Lord) need to be drawn programmatically on canvas or sourced from OpenGameArt. Keep them simple but characterful — 48×64 to match the style.

### Attribution
Sprites by Jordan Irwin (AntumDeluge), Svetlana Kushnariova, Lanea Zimmerman. Licensed under CC-BY 3.0 / CC-BY 4.0 / OGA-BY 3.0. Attribution must appear in the game credits.

---

## Sound Effects

All sounds generated procedurally via Web Audio API (no audio files). See `sound-effects-preview.html` for interactive demos.

### Training Phase
- **Correct answer:** Happy ascending chime (sine C5→G5 + triangle sparkle)
- **Wrong answer:** Soft descending buzz (square 220→140Hz)
- **Streak bonus:** Rapid ascending arpeggio (C5-E5-G5-C6) with shimmer
- **Power-up evolution:** Dramatic sweep into fanfare (sawtooth sweep + square melody)

### Battle Phase
- **Angel attack hit:** Sharp metallic impact (noise burst + triangle ring)
- **Divine strike (3x):** Holy slash whoosh + impact + angelic chord
- **Boss attack:** Dark rumble with crackle (sawtooth + filtered noise)
- **Super power activated:** Magical charge-up shimmer with crescendo

### UI & Transitions
- **Timer tick:** Subtle sine ping (880Hz, 80ms)
- **Timer urgent (<3s):** Fast double square beep (1200Hz)
- **Phase transition:** Whoosh sweep (sawtooth + bandpass filter)
- **Battle start:** Dramatic horn + drum roll
- **Victory fanfare:** Triumphant melody (C-E-G-C ascending)
- **Defeat:** Sad descending notes (C5-B4-A4-E4)

---

## Technical Requirements

### Single HTML File
- Everything in ONE `.html` file: HTML, CSS, JavaScript, sprite data (base64), and procedural audio
- No external dependencies, no CDN imports
- Should work offline once opened

### Rendering
- HTML5 `<canvas>` element for the game area (800×500 pixels, scales responsively)
- Sprite sheets embedded as base64 data URIs, drawn via `drawImage` with frame extraction
- Procedural backgrounds drawn per-level using canvas gradients, shapes, and particle systems
- `image-rendering: pixelated` for crisp pixel scaling

### Animations
- Smooth transitions between angel evolution stages
- Screen flash on correct answers
- Screen shake on attacks during boss fight
- Particle effects: fireflies (L1), snowfall (L2), bubbles (L3), embers (L4), dark energy (L5)
- Health bars animate smoothly when changing
- Boss entrance animations per level

### Responsive Design
- Game canvas scales to fit the browser window while maintaining aspect ratio
- Answer buttons should be large and easy to click (kid-friendly)
- Works on both desktop and tablet browsers

---

## Game Balance Summary

| Parameter | Level 1 | Level 2 | Level 3 | Level 4 | Level 5 |
|---|---|---|---|---|---|
| Training questions | 6 | 9 | 12 | 14 | 15 |
| Battle questions | 5 | 6 | 7 | 8 | 10 |
| Multiplication table | ×2 | ×3 | ×4 | ×5 | ×6 |
| Battle Q priority | Wrong answers first, then random from table |||||
| Training timer | 15s | 15s | 15s | 15s | 15s |
| Battle timer | 10s | 9s | 8s | 8s | 7s |
| Boss HP (normal hits) | 4 | 5 | 6 | 7 | 9 |
| Angel HP (boss hits) | 5 | 5 | 4 | 4 | 4 |
| Target win rate | ~85% | ~80% | ~75% | ~75% | ~70% |
| Approx level time | 3 min | 4 min | 5 min | 5 min | 6 min |

---

## State Management

All game state lives in JavaScript variables (no localStorage). Key state:

```
// Campaign state
currentLevel: 1–5
totalScore: number
levelsCompleted: [boolean x 5]

// Per-level state (resets each level)
gamePhase: "title" | "level-intro" | "training" | "battle" | "victory" | "defeat" | "campaign-complete"
angelLevel: 0–4 (evolution stage)
powerBar: 0–100 (percentage)
streakCount: number
angelHP: number
bossHP: number
levelScore: number
questionsAnswered: number
correctAnswers: number
battleQuestionsRemaining: number
wrongAnswers: [{a, b}]  // questions answered incorrectly during training — used to prioritise battle questions
```

---

## Implementation Notes for Claude Code

1. **Game loop and canvas setup** — render cycle, responsive scaling, state machine
2. **Sprite loading** — decode base64 sprites, frame extraction from sprite sheets
3. **Background renderer** — procedural backdrop per level (gradients, shapes, particles)
4. **Question generator** — single table per level (×2 through ×6), plausible distractors, wrong-answer tracking for battle priority queue
5. **Phase 1: Training** — power bar, evolution, streak bonuses, timer
6. **Phase 2: Boss fight** — turn-based combat, speed bonuses, super powers, boss AI
7. **Campaign flow** — level progression, intro screens, campaign complete screen
8. **Animations** — screen flash, shake, particles per world, health bar transitions
9. **Sound effects** — procedural Web Audio API (see sound-effects-preview.html)
10. **Boss characters** — Devil Queen and Violet Avenger from sprites; Shadow Wolf, Frost Witch, Kraken Lord drawn programmatically
11. **Polish** — title screen, transitions, score tracking, credits with attribution
12. **Test and balance** — ensure a kid getting ~70% right can beat most levels

### Key Principles
- **Fun first** — this should feel like a real game, not a quiz
- **Encouraging, never punishing** — wrong answers are gentle, no scary death scenes
- **Juice it up** — screen shakes, flashes, particles, and sounds make every correct answer feel AMAZING
- **Varied worlds** — each level should feel like a new adventure with distinct visuals and mood
- **One file** — everything must be self-contained in a single HTML file

### Reference Files
- `character-art-preview.html` — sprite preview with all angel stages and boss sprites
- `question-ui-mockup.html` — interactive UI mockup for training and battle phases
- `sound-effects-preview.html` — interactive sound effects demo (all procedural Web Audio API)
