# Angel vs Devil — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file HTML multiplication game with 5 levels, sprite-based characters, procedural backgrounds, particle effects, Web Audio sounds, and responsive scaling.

**Architecture:** Single `<canvas>` + DOM overlay. One `requestAnimationFrame` game loop, state machine for phases. Sprites decoded from base64 at startup. All code, styles, sprites, and sounds embedded in one HTML file.

**Tech Stack:** Vanilla HTML/CSS/JS, Canvas 2D API, Web Audio API. No dependencies.

**Critical sprite detail:** Sprite sheets are 144x256 pixels (3 cols x 4 rows). Each frame is **48x64 pixels** (not 16x16 as some docs say). Frame extraction: `drawImage(img, col*48, row*64, 48, 64, dx, dy, scaledW, scaledH)`. The kraken sprite is 96x128 (3x4 grid of 32x32 frames).

**Reference files:**
- `sprites_b64.json` — base64 sprite data for angel/devil characters
- `character-art-preview.html` — working sprite extraction + Seraphim glow effect code
- `question-ui-mockup.html` — training/battle UI layout reference
- `sound-effects-preview.html` — all 14 Web Audio API sound functions (copy directly)
- `docs/superpowers/specs/2026-04-03-angel-vs-devil-game-design.md` — full game design spec

---

### Task 1: Download Boss Sprites and Prepare Base64 Data

**Files:**
- Modify: `sprites_b64.json` (add werewolf, witch, kraken keys)

- [ ] **Step 1: Download the werewolf SWEN sprite**

```bash
curl -L -o /tmp/werewolf-SWEN.png "https://opengameart.org/sites/default/files/werewolf-SWEN.png"
```

If the direct URL doesn't work, visit https://opengameart.org/content/werewolf-cabbit and find the download link for `werewolf-SWEN.png`.

- [ ] **Step 2: Download the witch sprite**

```bash
# Download the zip and extract the SWEN variant
curl -L -o /tmp/witch.zip "https://opengameart.org/sites/default/files/witch_on_broomstick-1.2.zip"
cd /tmp && unzip -o witch.zip
# Look for the 48x64 SWEN PNG
find /tmp -name "*witch*SWEN*48*" -o -name "*witch*swen*48*" -o -name "*witch*48x64*SWEN*" | head -5
```

Locate the 48x64 SWEN PNG from the extracted files.

- [ ] **Step 3: Download the kraken sprite**

```bash
curl -L -o /tmp/kraken.zip "https://opengameart.org/sites/default/files/kraken.zip"
cd /tmp && unzip -o kraken.zip
# Find the SWEN PNG. It's 96x128.
find /tmp -name "*kraken*" -iname "*.png" | head -10
```

Locate the sprite PNG (may be NESW or SWEN — check which).

- [ ] **Step 4: Convert all three sprites to base64 and add to sprites_b64.json**

```bash
python3 -c "
import json, base64, sys

with open('/Users/ionut/Documents/GitHub/angel/sprites_b64.json') as f:
    data = json.load(f)

# Add each new sprite as data URI (adjust paths from steps 1-3)
for key, path in [('werewolf', '/tmp/werewolf-SWEN.png'), ('witch', '/tmp/WITCH_PATH_HERE'), ('kraken', '/tmp/KRAKEN_PATH_HERE')]:
    with open(path, 'rb') as img:
        b64 = base64.b64encode(img.read()).decode()
        data[key] = f'data:image/png;base64,{b64}'

with open('/Users/ionut/Documents/GitHub/angel/sprites_b64.json', 'w') as f:
    json.dump(data, f)

print('Added keys:', [k for k in data.keys()])
"
```

Adjust the file paths based on what you found in steps 2 and 3. Verify the existing sprites have `data:image/png;base64,` prefix — if they don't (just raw base64), match that format for consistency.

- [ ] **Step 5: Verify sprite dimensions**

```bash
python3 -c "
import json, base64, struct
d = json.load(open('/Users/ionut/Documents/GitHub/angel/sprites_b64.json'))
for k in ['werewolf', 'witch', 'kraken']:
    val = d[k]
    b64 = val.split(',')[1] if ',' in val else val
    data = base64.b64decode(b64)
    w = struct.unpack('>I', data[16:20])[0]
    h = struct.unpack('>I', data[20:24])[0]
    print(f'{k}: {w}x{h}')
"
```

Expected: werewolf 144x256 (48x64 frames), witch 144x256 (48x64 frames), kraken 96x128 (32x32 frames). If kraken dimensions differ, update the frame size logic in Task 3's `extractFrames` function accordingly.

- [ ] **Step 6: Commit**

```bash
git add sprites_b64.json
git commit -m "feat: add werewolf, witch, kraken boss sprites as base64"
```

---

### Task 2: HTML Scaffold, Canvas Setup, Responsive Scaling

**Files:**
- Create: `angel-vs-devil.html`

- [ ] **Step 1: Create the HTML file with canvas, DOM overlay, and responsive scaling**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Angel vs Devil — Math Battle Campaign</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
#game-container {
  position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
}
#game-canvas {
  display: block; image-rendering: pixelated; image-rendering: crisp-edges;
}
#ui-overlay {
  position: absolute; top: 0; left: 0; width: 800px; height: 500px;
  pointer-events: none; font-family: 'Segoe UI', system-ui, sans-serif;
}
#ui-overlay * { pointer-events: auto; }

/* HUD */
.hud { position: absolute; top: 0; left: 0; right: 0; padding: 10px 16px; display: flex; justify-content: space-between; align-items: flex-start; pointer-events: none; }
.hud-left, .hud-right { display: flex; flex-direction: column; gap: 4px; }
.hud-center { position: absolute; top: 8px; left: 50%; transform: translateX(-50%); }

/* Power bar */
.power-bar { width: 180px; height: 16px; background: #111; border-radius: 8px; border: 2px solid #333; overflow: hidden; position: relative; }
.power-fill { height: 100%; border-radius: 6px; transition: width 0.5s ease; background: linear-gradient(90deg, #2266ff, #44aaff, #88ddff); }
.power-label { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: 10px; font-weight: bold; color: #fff; text-shadow: 0 1px 2px #000; }

/* HP bars */
.hp-bar { width: 150px; height: 14px; background: #111; border-radius: 7px; border: 2px solid #333; overflow: hidden; position: relative; }
.hp-fill { height: 100%; border-radius: 5px; transition: width 0.4s ease; }
.hp-fill.angel { background: linear-gradient(90deg, #22aa44, #44dd66); }
.hp-fill.boss { background: linear-gradient(90deg, #dd2222, #ff4444); }
.hp-label { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: 9px; font-weight: bold; color: #fff; text-shadow: 0 1px 2px #000; }

/* Timer */
.timer-ring { width: 46px; height: 46px; position: relative; }
.timer-ring svg { width: 100%; height: 100%; }
.timer-text { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: 18px; font-weight: bold; color: #fff; }

/* Question area */
.question-area {
  position: absolute; bottom: 0; left: 0; right: 0;
  padding: 12px 16px 16px;
  background: linear-gradient(180deg, rgba(10,10,30,0) 0%, rgba(10,10,30,0.95) 30%);
  display: flex; flex-direction: column; align-items: center;
}
.question-text {
  font-size: 32px; font-weight: bold; color: #fff;
  margin-bottom: 10px; letter-spacing: 2px;
  text-shadow: 0 0 20px rgba(255,255,255,0.3);
}
.question-text .q-mark { color: #ffd700; }
.answers {
  display: flex; gap: 10px; justify-content: center; flex-wrap: wrap;
}
.answer-btn {
  min-width: 120px; height: 52px; border-radius: 10px; font-size: 22px; font-weight: bold;
  cursor: pointer; border: 3px solid #444;
  background: linear-gradient(180deg, #2a2a50, #1a1a40); color: #fff;
  transition: all 0.15s; text-shadow: 0 1px 3px rgba(0,0,0,0.5);
  -webkit-tap-highlight-color: transparent; user-select: none;
}
.answer-btn:hover { border-color: #88aaff; transform: translateY(-2px); box-shadow: 0 4px 15px rgba(100,150,255,0.3); }
.answer-btn:active { transform: scale(0.95); }
.answer-btn.correct { border-color: #44dd44; background: linear-gradient(180deg, #225522, #114411); color: #44ff44; box-shadow: 0 0 20px rgba(68,221,68,0.4); }
.answer-btn.wrong { border-color: #dd4444; background: linear-gradient(180deg, #442222, #331111); color: #ff6666; }

/* Super power selection */
.super-select {
  position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
  display: flex; gap: 20px; z-index: 10;
}
.super-btn {
  width: 100px; height: 100px; border-radius: 16px; border: 3px solid #ffd700;
  background: rgba(20,20,50,0.9); cursor: pointer; display: flex; flex-direction: column;
  align-items: center; justify-content: center; gap: 6px; color: #ffd700;
  font-size: 12px; font-weight: bold; transition: all 0.2s;
}
.super-btn:hover { transform: scale(1.1); box-shadow: 0 0 25px rgba(255,215,0,0.5); }
.super-icon { font-size: 36px; }

/* Floating text */
.float-text {
  position: absolute; pointer-events: none; font-weight: bold;
  animation: float-up 1s ease forwards;
}
@keyframes float-up {
  0% { opacity: 0; transform: translate(-50%, 0) scale(0.5); }
  20% { opacity: 1; transform: translate(-50%, -10px) scale(1.2); }
  100% { opacity: 0; transform: translate(-50%, -40px) scale(1); }
}

/* Screens (title, level-intro, victory, defeat, campaign-complete) */
.screen-overlay {
  position: absolute; inset: 0; display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  background: rgba(0,0,0,0.85); z-index: 20;
  transition: opacity 0.3s;
}
.screen-overlay.hidden { opacity: 0; pointer-events: none; }
.game-title {
  font-size: 42px; font-weight: bold; color: #ffd700;
  text-shadow: 0 0 30px rgba(255,215,0,0.5), 0 2px 4px rgba(0,0,0,0.8);
  letter-spacing: 3px;
}
.game-btn {
  margin-top: 24px; padding: 14px 40px; border-radius: 12px;
  border: 3px solid #ffd700; background: linear-gradient(180deg, #3a2a10, #2a1a05);
  color: #ffd700; font-size: 18px; font-weight: bold; cursor: pointer;
  transition: all 0.2s; letter-spacing: 1px;
}
.game-btn:hover { background: linear-gradient(180deg, #4a3a20, #3a2a10); transform: scale(1.05); box-shadow: 0 0 20px rgba(255,215,0,0.3); }

.level-name {
  font-size: 36px; color: #fff; font-weight: bold; letter-spacing: 2px;
  text-shadow: 0 0 20px rgba(255,255,255,0.3);
}
.flavor-text { color: #aaa; font-size: 16px; margin-top: 8px; font-style: italic; }

.stats-panel {
  background: rgba(20,20,50,0.8); border: 2px solid #333; border-radius: 12px;
  padding: 16px 24px; margin-top: 16px; text-align: left; color: #ccc; font-size: 14px;
  line-height: 1.8;
}
.stats-panel .label { color: #888; }
.stats-panel .value { color: #ffd700; font-weight: bold; }

.progress-dots { display: flex; gap: 8px; margin-top: 12px; }
.progress-dot {
  width: 14px; height: 14px; border-radius: 50%; border: 2px solid #555;
  background: #222; transition: all 0.3s;
}
.progress-dot.complete { background: #ffd700; border-color: #ffd700; box-shadow: 0 0 8px rgba(255,215,0,0.5); }
.progress-dot.current { border-color: #88aaff; }

/* Label styles */
.hud-label { font-size: 11px; font-weight: bold; text-shadow: 0 1px 3px rgba(0,0,0,0.8); }
.streak-label { color: #ffd700; font-size: 13px; font-weight: bold; }
.score-label { color: #aaa; font-size: 12px; text-align: right; }

/* Responsive: phone landscape */
@media (max-width: 700px) and (orientation: landscape) {
  .answers { flex-wrap: wrap; max-width: 300px; }
  .answer-btn { min-width: 130px; flex: 1 1 45%; }
}
</style>
</head>
<body>
<div id="game-container">
  <canvas id="game-canvas" width="800" height="500"></canvas>
  <div id="ui-overlay"></div>
</div>

<script>
// ============================================================
// ANGEL VS DEVIL — MATH BATTLE CAMPAIGN
// ============================================================

// --- SPRITE DATA (embedded base64) ---
const SPRITE_DATA = /* WILL BE FILLED IN TASK 3 */ {};

// --- LEVEL CONFIG ---
const LEVELS = [
  { name: 'Enchanted Forest', table: 2, trainingQs: 6, battleQs: 5, battleTimer: 10,
    boss: { name: 'Shadow Wolf', sprite: 'werewolf', color: '#8844aa' },
    flavor: 'The Shadow Wolf awaits...' },
  { name: 'Frozen Peaks', table: 3, trainingQs: 9, battleQs: 6, battleTimer: 9,
    boss: { name: 'Frost Witch', sprite: 'witch', color: '#44aadd' },
    flavor: 'The Frost Witch awaits...' },
  { name: 'Sunken Ruins', table: 4, trainingQs: 12, battleQs: 7, battleTimer: 8,
    boss: { name: 'Kraken Lord', sprite: 'kraken', color: '#2288aa' },
    flavor: 'The Kraken Lord awaits...' },
  { name: 'Volcanic Wastes', table: 5, trainingQs: 14, battleQs: 8, battleTimer: 8,
    boss: { name: 'Devil Queen', sprite: 'devil', color: '#dd4422' },
    flavor: 'The Devil Queen awaits...' },
  { name: 'The Dark Throne', table: 6, trainingQs: 15, battleQs: 10, battleTimer: 7,
    boss: { name: 'Violet Avenger', sprite: 'violet', color: '#8833cc' },
    flavor: 'The Violet Avenger awaits...' },
];

const EVOLUTION_THRESHOLDS = [
  [1, 3, 4, 6],   // Level 1
  [2, 4, 6, 9],   // Level 2
  [2, 5, 8, 12],  // Level 3
  [3, 6, 10, 14], // Level 4
  [3, 7, 11, 15], // Level 5
];

const BOSS_HP = [4, 5, 6, 7, 9];
const ANGEL_HP = [5, 5, 4, 4, 4];
const LEVEL_MULTIPLIER = [1.0, 1.2, 1.4, 1.6, 2.0];

// --- GAME STATE ---
let state = {};

function resetCampaignState() {
  state = {
    currentLevel: 0,
    totalScore: 0,
    levelsCompleted: [false, false, false, false, false],
    gamePhase: 'title',
    // Per-level (reset in resetLevelState)
    angelStage: 0, powerBar: 0, streakCount: 0,
    angelHP: 0, bossHP: 0,
    levelScore: 0, questionsAnswered: 0, correctAnswers: 0,
    wrongAnswers: [], battleQuestions: [], battleQIndex: 0,
    activeSuperPower: null, hasShield: false,
    questionStartTime: 0, totalAnswerTime: 0,
    fastestAnswer: Infinity, bestStreak: 0,
    trainingQuestions: [], trainingQIndex: 0,
    // Animation state
    screenShake: { x: 0, y: 0, intensity: 0, duration: 0 },
    screenFlash: { alpha: 0 },
    fadeAlpha: 0, fadeDirection: 0, fadeCallback: null,
    particles: [],
    floatTexts: [],
    angelPos: { x: 150, y: 260 }, bossPos: { x: 580, y: 240 },
    angelBob: 0, bossBob: 0,
    animationQueue: [],
    currentAnimation: null,
  };
}

function resetLevelState() {
  const lvl = LEVELS[state.currentLevel];
  state.angelStage = 0;
  state.powerBar = 0;
  state.streakCount = 0;
  state.angelHP = ANGEL_HP[state.currentLevel];
  state.bossHP = BOSS_HP[state.currentLevel];
  state.levelScore = 0;
  state.questionsAnswered = 0;
  state.correctAnswers = 0;
  state.wrongAnswers = [];
  state.battleQuestions = [];
  state.battleQIndex = 0;
  state.activeSuperPower = null;
  state.hasShield = false;
  state.questionStartTime = 0;
  state.totalAnswerTime = 0;
  state.fastestAnswer = Infinity;
  state.bestStreak = 0;
  state.trainingQuestions = generateTrainingQuestions(lvl.table, lvl.trainingQs);
  state.trainingQIndex = 0;
  state.particles = [];
  state.floatTexts = [];
  state.animationQueue = [];
  state.currentAnimation = null;
}

// --- CANVAS & SCALING ---
const canvas = document.getElementById('game-canvas');
const ctx = canvas.getContext('2d');
const overlay = document.getElementById('ui-overlay');
const container = document.getElementById('game-container');
const W = 800, H = 500;
let scale = 1;

function resize() {
  const vw = window.innerWidth, vh = window.innerHeight;
  scale = Math.min(vw / W, vh / H);
  const sw = W * scale, sh = H * scale;
  container.style.width = sw + 'px';
  container.style.height = sh + 'px';
  canvas.style.width = sw + 'px';
  canvas.style.height = sh + 'px';
  overlay.style.transform = 'scale(' + scale + ')';
  overlay.style.transformOrigin = 'top left';
  overlay.style.width = W + 'px';
  overlay.style.height = H + 'px';
}
window.addEventListener('resize', resize);
resize();

// --- PLACEHOLDER FUNCTIONS (filled in by later tasks) ---

function generateTrainingQuestions(table, count) { return []; }
function update(dt) {}
function render() {}

// --- GAME LOOP ---
let lastTime = 0;
function gameLoop(time) {
  const dt = Math.min((time - lastTime) / 1000, 0.1);
  lastTime = time;
  update(dt);
  render();
  requestAnimationFrame(gameLoop);
}

// --- INIT ---
resetCampaignState();
requestAnimationFrame(gameLoop);

</script>
</body>
</html>
```

- [ ] **Step 2: Open in Chrome and screenshot to verify blank canvas with proper scaling**

```bash
# Open in Chrome
open -a "Google Chrome" /Users/ionut/Documents/GitHub/angel/angel-vs-devil.html
# Wait for it to load, then screenshot
sleep 2
screencapture -l $(osascript -e 'tell app "Google Chrome" to id of window 1') /tmp/playtest-task2.png
```

Verify: black canvas filling the browser window, maintaining 16:10 aspect ratio.

- [ ] **Step 3: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: HTML scaffold with canvas, DOM overlay, responsive scaling"
```

---

### Task 3: Sprite Loading System

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Embed sprite data and build sprite loader**

Replace the `SPRITE_DATA` placeholder with the actual base64 data from `sprites_b64.json`. Then add the sprite loading and frame extraction code.

Read `sprites_b64.json` and embed each value as a property of `SPRITE_DATA`. The keys are: `fallen1`, `fallen2`, `dark`, `angel`, `archangel`, `devil`, `violet`, `werewolf`, `witch`, `kraken`.

Add this code after the `SPRITE_DATA` object:

```javascript
// --- SPRITE SYSTEM ---
const spriteImages = {};
const spriteFrames = {}; // spriteFrames[key][row][col] = offscreen canvas

async function loadSprites() {
  const promises = Object.entries(SPRITE_DATA).map(([key, src]) => {
    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => {
        spriteImages[key] = img;
        extractFrames(key, img);
        resolve();
      };
      img.src = src;
    });
  });
  await Promise.all(promises);
}

function extractFrames(key, img) {
  // Determine frame size: kraken is 96x128 (32x32 frames), all others are 144x256 (48x64 frames)
  const fw = key === 'kraken' ? 32 : 48;
  const fh = key === 'kraken' ? 32 : 64;
  const cols = 3, rows = 4;
  spriteFrames[key] = [];
  for (let r = 0; r < rows; r++) {
    spriteFrames[key][r] = [];
    for (let c = 0; c < cols; c++) {
      const offscreen = document.createElement('canvas');
      offscreen.width = fw;
      offscreen.height = fh;
      const octx = offscreen.getContext('2d');
      octx.imageSmoothingEnabled = false;
      octx.drawImage(img, c * fw, r * fh, fw, fh, 0, 0, fw, fh);
      spriteFrames[key][r][c] = offscreen;
    }
  }
}

// SWEN = row 0: South, row 1: West, row 2: East, row 3: North
const DIR = { S: 0, W: 1, E: 2, N: 3 };

function drawSprite(key, direction, frameIdx, x, y, scaleVal, options) {
  // options: { opacity, tint, glow }
  const fw = key === 'kraken' ? 32 : 48;
  const fh = key === 'kraken' ? 32 : 64;
  const sw = fw * scaleVal;
  const sh = fh * scaleVal;
  const frame = spriteFrames[key]?.[direction]?.[frameIdx % 3];
  if (!frame) return;

  ctx.save();
  if (options?.opacity !== undefined) ctx.globalAlpha = options.opacity;

  // Draw to temp canvas for effects
  if (options?.tint || options?.glow) {
    const tmp = document.createElement('canvas');
    tmp.width = fw; tmp.height = fh;
    const tc = tmp.getContext('2d');
    tc.imageSmoothingEnabled = false;
    tc.drawImage(frame, 0, 0);

    if (options.tint) {
      tc.globalCompositeOperation = 'source-atop';
      tc.fillStyle = options.tint;
      tc.fillRect(0, 0, fw, fh);
    }
    if (options.glow) {
      tc.globalCompositeOperation = 'lighter';
      tc.fillStyle = options.glow;
      tc.fillRect(0, 0, fw, fh);
    }
    tc.globalCompositeOperation = 'source-over';
    ctx.drawImage(tmp, x, y, sw, sh);
  } else {
    ctx.drawImage(frame, x, y, sw, sh);
  }
  ctx.restore();
}

// Angel sprite key based on evolution stage
function getAngelSprite() {
  if (state.angelStage <= 1) return 'fallen2';
  if (state.angelStage === 2) return 'angel';
  return 'archangel'; // stages 3 and 4
}

function getAngelSpriteOptions() {
  const opts = {};
  if (state.angelStage === 0) opts.opacity = 0.5;
  if (state.angelStage === 4) {
    opts.tint = 'rgba(255,215,0,0.35)';
    opts.glow = 'rgba(255,230,100,0.18)';
  }
  return opts;
}
```

- [ ] **Step 2: Update init to load sprites before starting game loop**

Replace the init section at the bottom:

```javascript
// --- INIT ---
resetCampaignState();
loadSprites().then(() => {
  resize();
  requestAnimationFrame(gameLoop);
});
```

- [ ] **Step 3: Add a temporary render test to draw the angel sprite**

Temporarily replace the `render()` function:

```javascript
function render() {
  ctx.fillStyle = '#0a0a1e';
  ctx.fillRect(0, 0, W, H);

  // Test: draw all angel stages in a row
  const stages = [
    { key: 'fallen2', opts: { opacity: 0.5 } },
    { key: 'fallen2', opts: {} },
    { key: 'angel', opts: {} },
    { key: 'archangel', opts: {} },
    { key: 'archangel', opts: { tint: 'rgba(255,215,0,0.35)', glow: 'rgba(255,230,100,0.18)' } },
  ];
  stages.forEach((s, i) => {
    drawSprite(s.key, DIR.S, 1, 50 + i * 140, 150, 2.5, s.opts);
  });

  // Test: draw boss sprites
  const bosses = ['werewolf', 'witch', 'kraken', 'devil', 'violet'];
  bosses.forEach((key, i) => {
    const sc = key === 'kraken' ? 3 : 2.5;
    drawSprite(key, DIR.S, 1, 50 + i * 140, 320, sc, {});
  });
}
```

- [ ] **Step 4: Screenshot and verify all sprites render correctly**

```bash
# Reload Chrome and screenshot
osascript -e 'tell app "Google Chrome" to tell active tab of window 1 to reload'
sleep 2
screencapture -l $(osascript -e 'tell app "Google Chrome" to id of window 1') /tmp/playtest-task3.png
```

Verify: 5 angel evolution stages visible in top row, 5 bosses in bottom row. Check: no missing sprites, correct scaling, pixelated rendering (not blurry), Seraphim has golden tint. Witch should display but may not have ice tint yet (that's fine).

- [ ] **Step 5: Remove the test render code** (revert render to empty function)

- [ ] **Step 6: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: sprite loading system with frame extraction and tinting"
```

---

### Task 4: Sound Effects System

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Add all 14 sound effects**

Copy the sound functions directly from `sound-effects-preview.html`. Add them inside the `<script>` tag, after the sprite system code:

```javascript
// --- SOUND EFFECTS (Web Audio API) ---
let audioCtx = null;
function getAudioCtx() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  return audioCtx;
}
```

Then copy all 14 functions verbatim from `sound-effects-preview.html`:
- `sfxCorrect()`, `sfxWrong()`, `sfxStreak()`, `sfxPowerUp()`
- `sfxAttackHit()`, `sfxDivineStrike()`, `sfxDevilAttack()`, `sfxSuperPower()`
- `sfxTimerTick()`, `sfxTimerUrgent()`, `sfxTransition()`, `sfxBattleStart()`
- `sfxVictory()`, `sfxDefeat()`

The only change: rename `getCtx()` calls to `getAudioCtx()` in each function (since we already use `ctx` for canvas).

- [ ] **Step 2: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: all 14 procedural sound effects via Web Audio API"
```

---

### Task 5: Question Generator

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement question generation with distractor logic**

Replace the placeholder `generateTrainingQuestions`:

```javascript
// --- QUESTION SYSTEM ---
function generateTrainingQuestions(table, count) {
  // Generate all possible questions for this table (table x 1..10)
  const all = [];
  for (let i = 1; i <= 10; i++) {
    all.push({ a: table, b: i, answer: table * i });
  }
  // Shuffle
  shuffleArray(all);
  // Ensure no two consecutive questions have the same multiplier (b)
  for (let i = 1; i < all.length; i++) {
    if (all[i].b === all[i - 1].b) {
      // Swap with next different one
      for (let j = i + 1; j < all.length; j++) {
        if (all[j].b !== all[i - 1].b) {
          [all[i], all[j]] = [all[j], all[i]];
          break;
        }
      }
    }
  }
  // Take 'count' questions, cycling if needed
  const questions = [];
  for (let i = 0; i < count; i++) {
    questions.push({ ...all[i % all.length] });
  }
  return questions;
}

function generateDistractors(table, correctAnswer) {
  const distractors = new Set();
  // Adjacent products: table * (b-1), table * (b+1), table * (b-2), table * (b+2)
  const b = correctAnswer / table;
  [b - 2, b - 1, b + 1, b + 2].forEach(n => {
    if (n >= 1 && n <= 10) distractors.add(table * n);
  });
  // Random offsets
  for (let offset = 1; offset <= 5; offset++) {
    distractors.add(correctAnswer + offset);
    distractors.add(correctAnswer - offset);
  }
  // Remove correct answer, negatives, zero
  distractors.delete(correctAnswer);
  const valid = [...distractors].filter(d => d > 0);
  shuffleArray(valid);
  return valid.slice(0, 3);
}

function makeQuestion(q) {
  const distractors = generateDistractors(q.a, q.answer);
  const options = [q.answer, ...distractors];
  shuffleArray(options);
  return { ...q, options };
}

function generateBattleQuestions(table, count, wrongAnswers) {
  const questions = [];
  // First: wrong answers from training (shuffled)
  const wrongs = [...wrongAnswers];
  shuffleArray(wrongs);
  wrongs.forEach(q => {
    if (questions.length < count) questions.push({ a: q.a, b: q.b, answer: q.a * q.b });
  });
  // Fill remaining with random from the table
  const pool = [];
  for (let i = 1; i <= 10; i++) pool.push({ a: table, b: i, answer: table * i });
  shuffleArray(pool);
  for (const q of pool) {
    if (questions.length >= count) break;
    // Avoid duplicates
    if (!questions.some(existing => existing.b === q.b)) questions.push(q);
  }
  // If still not enough, allow duplicates
  while (questions.length < count) {
    shuffleArray(pool);
    questions.push({ ...pool[0] });
  }
  return questions;
}

function shuffleArray(arr) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}
```

- [ ] **Step 2: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: question generator with distractors and wrong-answer priority"
```

---

### Task 6: Particle System & Screen Effects

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement particle system and screen effects**

```javascript
// --- PARTICLE SYSTEM ---
function addParticle(p) {
  state.particles.push({
    x: p.x, y: p.y,
    vx: p.vx || 0, vy: p.vy || 0,
    size: p.size || 3,
    color: p.color || '#fff',
    alpha: p.alpha || 1,
    life: p.life || 1,
    maxLife: p.life || 1,
    gravity: p.gravity || 0,
    shrink: p.shrink !== false,
  });
}

function addParticleBurst(x, y, count, color, speed, life) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2;
    const spd = Math.random() * speed;
    addParticle({
      x, y,
      vx: Math.cos(angle) * spd,
      vy: Math.sin(angle) * spd,
      color,
      size: 2 + Math.random() * 4,
      life: life * (0.5 + Math.random() * 0.5),
    });
  }
}

function updateParticles(dt) {
  for (let i = state.particles.length - 1; i >= 0; i--) {
    const p = state.particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vy += p.gravity * dt;
    p.life -= dt;
    p.alpha = Math.max(0, p.life / p.maxLife);
    if (p.shrink) p.size *= (1 - dt * 0.5);
    if (p.life <= 0) state.particles.splice(i, 1);
  }
}

function renderParticles() {
  state.particles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = p.alpha;
    ctx.fillStyle = p.color;
    ctx.shadowColor = p.color;
    ctx.shadowBlur = p.size * 2;
    ctx.fillRect(p.x - p.size / 2, p.y - p.size / 2, p.size, p.size);
    ctx.restore();
  });
}

// --- SCREEN EFFECTS ---
function triggerScreenShake(intensity, duration) {
  state.screenShake.intensity = intensity;
  state.screenShake.duration = duration;
}

function triggerScreenFlash(alpha) {
  state.screenFlash.alpha = alpha || 0.4;
}

function updateScreenEffects(dt) {
  // Shake
  if (state.screenShake.duration > 0) {
    state.screenShake.duration -= dt;
    const i = state.screenShake.intensity;
    state.screenShake.x = (Math.random() - 0.5) * i * 2;
    state.screenShake.y = (Math.random() - 0.5) * i * 2;
  } else {
    state.screenShake.x = 0;
    state.screenShake.y = 0;
  }
  // Flash
  if (state.screenFlash.alpha > 0) {
    state.screenFlash.alpha = Math.max(0, state.screenFlash.alpha - dt * 3);
  }
}

function renderScreenFlash() {
  if (state.screenFlash.alpha > 0) {
    ctx.save();
    ctx.globalAlpha = state.screenFlash.alpha;
    ctx.fillStyle = '#fff';
    ctx.fillRect(0, 0, W, H);
    ctx.restore();
  }
}

// --- FADE TRANSITIONS ---
function startFade(direction, duration, callback) {
  state.fadeAlpha = direction === 'out' ? 0 : 1;
  state.fadeDirection = direction === 'out' ? 1 : -1;
  state.fadeDuration = duration || 0.3;
  state.fadeSpeed = 1 / state.fadeDuration;
  state.fadeCallback = callback;
}

function updateFade(dt) {
  if (state.fadeDirection === 0) return;
  state.fadeAlpha += state.fadeDirection * state.fadeSpeed * dt;
  if (state.fadeAlpha >= 1) {
    state.fadeAlpha = 1;
    state.fadeDirection = 0;
    if (state.fadeCallback) { state.fadeCallback(); state.fadeCallback = null; }
  } else if (state.fadeAlpha <= 0) {
    state.fadeAlpha = 0;
    state.fadeDirection = 0;
    if (state.fadeCallback) { state.fadeCallback(); state.fadeCallback = null; }
  }
}

function renderFade() {
  if (state.fadeAlpha > 0) {
    ctx.save();
    ctx.globalAlpha = state.fadeAlpha;
    ctx.fillStyle = '#000';
    ctx.fillRect(0, 0, W, H);
    ctx.restore();
  }
}

// --- FLOATING TEXT ---
function addFloatText(text, x, y, color, size) {
  state.floatTexts.push({ text, x, y, color: color || '#ffd700', size: size || 24, life: 1, maxLife: 1 });
}

function updateFloatTexts(dt) {
  for (let i = state.floatTexts.length - 1; i >= 0; i--) {
    const ft = state.floatTexts[i];
    ft.y -= 30 * dt;
    ft.life -= dt;
    if (ft.life <= 0) state.floatTexts.splice(i, 1);
  }
}

function renderFloatTexts() {
  state.floatTexts.forEach(ft => {
    const progress = 1 - ft.life / ft.maxLife;
    const alpha = progress < 0.2 ? progress / 0.2 : (1 - progress) / 0.8;
    const sc = progress < 0.2 ? 0.5 + progress * 2.5 : 1;
    ctx.save();
    ctx.globalAlpha = Math.max(0, alpha);
    ctx.font = 'bold ' + Math.round(ft.size * sc) + 'px "Segoe UI", system-ui, sans-serif';
    ctx.textAlign = 'center';
    ctx.fillStyle = ft.color;
    ctx.shadowColor = ft.color;
    ctx.shadowBlur = 10;
    ctx.fillText(ft.text, ft.x, ft.y);
    ctx.restore();
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: particle system, screen shake/flash, fade transitions, floating text"
```

---

### Task 7: Background Renderer (All 5 Levels)

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement procedural backgrounds for all 5 levels**

```javascript
// --- BACKGROUND SYSTEM ---
let bgCache = null;
let bgCacheLevel = -1;
let bgCacheBattle = false;

// Per-level ambient particles (persistent, not in state.particles)
let ambientParticles = [];

function initAmbientParticles(level) {
  ambientParticles = [];
  const count = [30, 50, 25, 40, 35][level];
  for (let i = 0; i < count; i++) {
    ambientParticles.push({
      x: Math.random() * W, y: Math.random() * H * 0.75,
      vx: (Math.random() - 0.5) * 20,
      vy: level === 2 ? 15 + Math.random() * 25 : // snow falls
          level === 3 ? -(5 + Math.random() * 10) : // bubbles rise
          level === 4 ? -(10 + Math.random() * 15) : // embers rise
          (Math.random() - 0.5) * 10,
      size: 1 + Math.random() * 3,
      alpha: 0.3 + Math.random() * 0.7,
      phase: Math.random() * Math.PI * 2,
    });
  }
}

function updateAmbientParticles(dt) {
  const level = state.currentLevel;
  ambientParticles.forEach(p => {
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.phase += dt * 2;

    // Wrap around
    if (p.x < -10) p.x = W + 10;
    if (p.x > W + 10) p.x = -10;
    if (p.y < -10) p.y = H + 10;
    if (p.y > H * 0.75 + 10) p.y = -10;
    if (p.y < -10 && (level === 3 || level === 4)) p.y = H * 0.75;

    // Fireflies pulse
    if (level === 0) p.alpha = 0.3 + Math.sin(p.phase) * 0.4;
  });
}

function renderAmbientParticles() {
  const level = state.currentLevel;
  const colors = ['#aaff44', '#ccddff', '#44ddff', '#ff8833', '#aa44ff'];
  ambientParticles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = p.alpha;
    ctx.fillStyle = colors[level];
    if (level === 0) { // Fireflies: glow
      ctx.shadowColor = '#aaff44';
      ctx.shadowBlur = 8;
    }
    if (level === 3) { // Bubbles: circles
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
      ctx.fill();
      ctx.globalAlpha = p.alpha * 0.5;
      ctx.fillStyle = '#fff';
      ctx.beginPath();
      ctx.arc(p.x - p.size * 0.3, p.y - p.size * 0.3, p.size * 0.3, 0, Math.PI * 2);
      ctx.fill();
    } else {
      ctx.fillRect(p.x - p.size / 2, p.y - p.size / 2, p.size, p.size);
    }
    ctx.restore();
  });
}

function drawBackground(level, isBattle) {
  // Check cache
  if (bgCache && bgCacheLevel === level && bgCacheBattle === isBattle) {
    ctx.drawImage(bgCache, 0, 0);
    return;
  }

  const offscreen = document.createElement('canvas');
  offscreen.width = W;
  offscreen.height = H;
  const bx = offscreen.getContext('2d');

  const groundY = H * 0.72;
  const darken = isBattle ? 0.7 : 1.0;

  if (level === 0) drawForestBg(bx, groundY, darken);
  else if (level === 1) drawPeaksBg(bx, groundY, darken);
  else if (level === 2) drawRuinsBg(bx, groundY, darken);
  else if (level === 3) drawVolcanoBg(bx, groundY, darken);
  else if (level === 4) drawThroneBg(bx, groundY, darken);

  bgCache = offscreen;
  bgCacheLevel = level;
  bgCacheBattle = isBattle;
  ctx.drawImage(offscreen, 0, 0);
}

function drawForestBg(bx, groundY, d) {
  // Sky
  const sky = bx.createLinearGradient(0, 0, 0, groundY);
  sky.addColorStop(0, lerpColor('#0a1628', '#000', 1 - d));
  sky.addColorStop(1, lerpColor('#1a3a2a', '#000', 1 - d));
  bx.fillStyle = sky;
  bx.fillRect(0, 0, W, groundY);

  // Moonlight
  const moon = bx.createRadialGradient(600, 60, 20, 600, 60, 200);
  moon.addColorStop(0, 'rgba(255,255,240,0.15)');
  moon.addColorStop(1, 'rgba(255,255,240,0)');
  bx.fillStyle = moon;
  bx.fillRect(0, 0, W, groundY);

  // Light shafts
  bx.save();
  bx.globalAlpha = 0.06 * d;
  bx.fillStyle = '#ffffcc';
  [[300, -20, 340, groundY, 280, groundY],
   [500, -20, 530, groundY, 480, groundY],
   [650, -20, 680, groundY, 630, groundY]].forEach(s => {
    bx.beginPath(); bx.moveTo(s[0], s[1]); bx.lineTo(s[2], s[3]); bx.lineTo(s[4], s[5]); bx.closePath(); bx.fill();
  });
  bx.restore();

  // Trees
  bx.fillStyle = '#0a1a0a';
  drawTree(bx, 50, groundY);
  drawTree(bx, 700, groundY);
  bx.fillStyle = '#0d1f0d';
  drawTree(bx, 150, groundY, 0.7);

  // Ground
  const ground = bx.createLinearGradient(0, groundY, 0, H);
  ground.addColorStop(0, '#1a3a1a');
  ground.addColorStop(1, '#0d2a0d');
  bx.fillStyle = ground;
  bx.fillRect(0, groundY, W, H - groundY);

  // Glowing mushrooms
  [[120, groundY + 20], [350, groundY + 30], [580, groundY + 15]].forEach(([mx, my]) => {
    const mg = bx.createRadialGradient(mx, my, 2, mx, my, 15);
    mg.addColorStop(0, 'rgba(100,255,100,0.4)');
    mg.addColorStop(1, 'rgba(100,255,100,0)');
    bx.fillStyle = mg;
    bx.fillRect(mx - 15, my - 15, 30, 30);
    bx.fillStyle = '#44aa44';
    bx.beginPath(); bx.ellipse(mx, my, 5, 3, 0, 0, Math.PI * 2); bx.fill();
  });

  // Stone ruin
  bx.fillStyle = 'rgba(80,80,70,0.3)';
  bx.fillRect(380, groundY - 60, 15, 60);
  bx.fillRect(420, groundY - 45, 12, 45);
  bx.fillRect(375, groundY - 65, 65, 8);
}

function drawTree(bx, x, groundY, s) {
  s = s || 1;
  // Trunk
  bx.fillRect(x - 5 * s, groundY - 80 * s, 10 * s, 80 * s);
  // Canopy (bezier blobs)
  bx.beginPath();
  bx.moveTo(x - 40 * s, groundY - 60 * s);
  bx.bezierCurveTo(x - 50 * s, groundY - 120 * s, x - 10 * s, groundY - 150 * s, x, groundY - 130 * s);
  bx.bezierCurveTo(x + 10 * s, groundY - 150 * s, x + 50 * s, groundY - 120 * s, x + 40 * s, groundY - 60 * s);
  bx.closePath();
  bx.fill();
}

function drawPeaksBg(bx, groundY, d) {
  // Sky
  const sky = bx.createLinearGradient(0, 0, 0, groundY);
  sky.addColorStop(0, '#0a0a2e');
  sky.addColorStop(0.5, '#1a1040');
  sky.addColorStop(1, '#1a2040');
  bx.fillStyle = sky;
  bx.fillRect(0, 0, W, groundY);

  // Aurora bands
  bx.save();
  for (let i = 0; i < 3; i++) {
    bx.globalAlpha = 0.08;
    const y = 40 + i * 30;
    const grad = bx.createLinearGradient(0, y, W, y);
    grad.addColorStop(0, i % 2 ? '#22ff88' : '#8844ff');
    grad.addColorStop(0.5, i % 2 ? '#44ffaa' : '#aa66ff');
    grad.addColorStop(1, i % 2 ? '#22ff88' : '#8844ff');
    bx.fillStyle = grad;
    bx.beginPath();
    bx.moveTo(0, y);
    for (let x = 0; x <= W; x += 10) {
      bx.lineTo(x, y + Math.sin(x * 0.01 + i * 2) * 15);
    }
    bx.lineTo(W, y + 20);
    bx.lineTo(0, y + 20);
    bx.closePath();
    bx.fill();
  }
  bx.restore();

  // Mountains
  bx.fillStyle = '#2a2a4a';
  bx.beginPath(); bx.moveTo(0, groundY); bx.lineTo(150, groundY - 180); bx.lineTo(300, groundY); bx.fill();
  bx.fillStyle = '#222244';
  bx.beginPath(); bx.moveTo(200, groundY); bx.lineTo(400, groundY - 220); bx.lineTo(550, groundY); bx.fill();
  bx.fillStyle = '#1a1a3a';
  bx.beginPath(); bx.moveTo(450, groundY); bx.lineTo(650, groundY - 160); bx.lineTo(800, groundY); bx.fill();

  // Snow caps
  bx.fillStyle = '#ddeeff';
  bx.beginPath(); bx.moveTo(140, groundY - 170); bx.lineTo(150, groundY - 180); bx.lineTo(160, groundY - 170); bx.fill();
  bx.beginPath(); bx.moveTo(385, groundY - 205); bx.lineTo(400, groundY - 220); bx.lineTo(415, groundY - 205); bx.fill();
  bx.beginPath(); bx.moveTo(640, groundY - 150); bx.lineTo(650, groundY - 160); bx.lineTo(660, groundY - 150); bx.fill();

  // Pine trees
  [[100, groundY], [500, groundY], [720, groundY]].forEach(([tx, ty]) => {
    bx.fillStyle = '#1a2a1a';
    bx.fillRect(tx - 3, ty - 30, 6, 30);
    for (let j = 0; j < 3; j++) {
      bx.beginPath();
      bx.moveTo(tx - 15 + j * 3, ty - 25 - j * 15);
      bx.lineTo(tx, ty - 50 - j * 15);
      bx.lineTo(tx + 15 - j * 3, ty - 25 - j * 15);
      bx.closePath();
      bx.fill();
    }
  });

  // Ground (snow)
  bx.fillStyle = '#ddeeff';
  bx.fillRect(0, groundY, W, H - groundY);
  bx.fillStyle = '#ccddf0';
  bx.fillRect(0, groundY + 10, W, H - groundY - 10);
}

function drawRuinsBg(bx, groundY, d) {
  // Underwater gradient
  const water = bx.createLinearGradient(0, 0, 0, H);
  water.addColorStop(0, '#0a2a3a');
  water.addColorStop(0.4, '#0a3a4a');
  water.addColorStop(1, '#082830');
  bx.fillStyle = water;
  bx.fillRect(0, 0, W, H);

  // Light shafts from above
  bx.save();
  bx.globalAlpha = 0.08;
  bx.fillStyle = '#88ddff';
  [[200, 250], [450, 350], [600, 300]].forEach(([sx, sw]) => {
    bx.beginPath();
    bx.moveTo(sx - 20, 0); bx.lineTo(sx + 30, 0);
    bx.lineTo(sx + sw / 2 + 40, H);
    bx.lineTo(sx - sw / 2 - 10, H);
    bx.closePath();
    bx.fill();
  });
  bx.restore();

  // Coral columns
  bx.fillStyle = '#1a4a5a';
  bx.fillRect(20, 100, 25, H - 100);
  bx.fillRect(W - 45, 80, 25, H - 80);

  // Coral details
  bx.fillStyle = '#2a5a6a';
  [[30, 120, 12], [35, 180, 8], [W - 38, 110, 10], [W - 30, 200, 7]].forEach(([cx, cy, cr]) => {
    bx.beginPath(); bx.arc(cx, cy, cr, 0, Math.PI * 2); bx.fill();
  });

  // Broken marble pillars
  bx.fillStyle = 'rgba(200,200,180,0.15)';
  bx.fillRect(250, groundY - 80, 18, 80);
  bx.fillRect(550, groundY - 60, 18, 60);
  bx.fillRect(540, groundY - 55, 40, 8); // broken top piece lying down

  // Ground
  bx.fillStyle = '#0a2020';
  bx.fillRect(0, groundY, W, H - groundY);
}

function drawVolcanoBg(bx, groundY, d) {
  // Sky
  const sky = bx.createLinearGradient(0, 0, 0, groundY);
  sky.addColorStop(0, '#1a0505');
  sky.addColorStop(0.5, '#2a0a0a');
  sky.addColorStop(1, '#3a1515');
  bx.fillStyle = sky;
  bx.fillRect(0, 0, W, groundY);

  // Crimson clouds
  bx.save();
  bx.globalAlpha = 0.15;
  [[100, 50, 120], [350, 30, 100], [600, 60, 80]].forEach(([cx, cy, r]) => {
    const cloud = bx.createRadialGradient(cx, cy, r * 0.3, cx, cy, r);
    cloud.addColorStop(0, '#ff4422');
    cloud.addColorStop(1, 'transparent');
    bx.fillStyle = cloud;
    bx.fillRect(cx - r, cy - r, r * 2, r * 2);
  });
  bx.restore();

  // Distant volcanos
  bx.fillStyle = '#2a1010';
  bx.beginPath(); bx.moveTo(0, groundY); bx.lineTo(100, groundY - 120); bx.lineTo(200, groundY); bx.fill();
  bx.fillStyle = '#331515';
  bx.beginPath(); bx.moveTo(550, groundY); bx.lineTo(700, groundY - 150); bx.lineTo(800, groundY); bx.fill();

  // Lava glow at volcano tops
  [[100, groundY - 120], [700, groundY - 150]].forEach(([vx, vy]) => {
    const glow = bx.createRadialGradient(vx, vy, 5, vx, vy, 30);
    glow.addColorStop(0, 'rgba(255,150,50,0.5)');
    glow.addColorStop(1, 'rgba(255,100,30,0)');
    bx.fillStyle = glow;
    bx.fillRect(vx - 30, vy - 30, 60, 60);
  });

  // Ground
  bx.fillStyle = '#1a0a05';
  bx.fillRect(0, groundY, W, H - groundY);

  // Lava rivers
  bx.save();
  bx.strokeStyle = '#ff8833';
  bx.lineWidth = 3;
  bx.shadowColor = '#ff6622';
  bx.shadowBlur = 10;
  [[50, groundY + 15, 200, groundY + 25],
   [350, groundY + 10, 500, groundY + 30],
   [600, groundY + 20, 750, groundY + 18]].forEach(([x1, y1, x2, y2]) => {
    bx.beginPath(); bx.moveTo(x1, y1);
    bx.quadraticCurveTo((x1 + x2) / 2, (y1 + y2) / 2 + 8, x2, y2);
    bx.stroke();
  });
  bx.restore();

  // Cracks
  bx.strokeStyle = '#331100';
  bx.lineWidth = 1;
  [[100, groundY + 5, 180, groundY + 35],
   [400, groundY + 8, 450, groundY + 40],
   [650, groundY + 3, 720, groundY + 38]].forEach(([x1, y1, x2, y2]) => {
    bx.beginPath(); bx.moveTo(x1, y1); bx.lineTo(x2, y2); bx.stroke();
  });
}

function drawThroneBg(bx, groundY, d) {
  // Void
  const void_ = bx.createLinearGradient(0, 0, 0, H);
  void_.addColorStop(0, '#0a0015');
  void_.addColorStop(0.5, '#150025');
  void_.addColorStop(1, '#0a0015');
  bx.fillStyle = void_;
  bx.fillRect(0, 0, W, H);

  // Warped stars
  bx.fillStyle = '#ffffff';
  for (let i = 0; i < 40; i++) {
    const sx = Math.random() * W;
    const sy = Math.random() * H * 0.6;
    const sz = Math.random() * 2;
    bx.globalAlpha = 0.2 + Math.random() * 0.4;
    bx.fillRect(sx, sy, sz, sz);
  }
  bx.globalAlpha = 1;

  // Castle silhouette
  bx.fillStyle = '#0a000a';
  // Main body
  bx.fillRect(300, 30, 200, 200);
  // Towers
  bx.fillRect(280, 50, 30, 180);
  bx.fillRect(490, 50, 30, 180);
  // Spires
  bx.beginPath(); bx.moveTo(295, 50); bx.lineTo(280, 10); bx.lineTo(310, 50); bx.fill();
  bx.beginPath(); bx.moveTo(505, 50); bx.lineTo(490, 10); bx.lineTo(520, 50); bx.fill();
  // Central spire
  bx.beginPath(); bx.moveTo(400, 30); bx.lineTo(385, -10); bx.lineTo(415, -10); bx.lineTo(400, 30); bx.fill();

  // Stained glass windows
  [[340, 80, '#8833cc'], [400, 80, '#cc3388'], [460, 80, '#3388cc']].forEach(([wx, wy, wc]) => {
    bx.fillStyle = wc;
    bx.globalAlpha = 0.2;
    bx.beginPath();
    bx.ellipse(wx, wy, 12, 20, 0, 0, Math.PI * 2);
    bx.fill();
    bx.globalAlpha = 1;
  });

  // Floor
  bx.fillStyle = '#0f000f';
  bx.fillRect(0, groundY, W, H - groundY);

  // Floor runes
  bx.save();
  bx.strokeStyle = 'rgba(136,51,204,0.15)';
  bx.lineWidth = 1;
  [[200, groundY + 20, 30], [400, groundY + 25, 40], [600, groundY + 15, 25]].forEach(([rx, ry, rr]) => {
    bx.beginPath(); bx.arc(rx, ry, rr, 0, Math.PI * 2); bx.stroke();
    // Inner pattern
    for (let a = 0; a < 6; a++) {
      const angle = a * Math.PI / 3;
      bx.beginPath();
      bx.moveTo(rx, ry);
      bx.lineTo(rx + Math.cos(angle) * rr, ry + Math.sin(angle) * rr * 0.5);
      bx.stroke();
    }
  });
  bx.restore();

  // Floating debris
  bx.fillStyle = '#1a0a2a';
  [[120, 180, 15, 10], [650, 150, 20, 12], [350, 250, 12, 8]].forEach(([dx, dy, dw, dh]) => {
    bx.save();
    bx.translate(dx + dw / 2, dy + dh / 2);
    bx.rotate(0.3);
    bx.fillRect(-dw / 2, -dh / 2, dw, dh);
    bx.restore();
  });
}

// Utility: lerp color for darkening in battle
function lerpColor(hex, target, t) {
  // Simple: just return hex for now, darkness is handled by overlay
  return hex;
}
```

- [ ] **Step 2: Screenshot to verify backgrounds**

Write a temporary render test that cycles through all 5 backgrounds:

```javascript
// Temporary: draw level background based on a counter
let testLevel = 0;
function render() {
  drawBackground(testLevel, false);
  renderAmbientParticles();
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 24px sans-serif';
  ctx.fillText('Level ' + (testLevel + 1) + ': ' + LEVELS[testLevel].name, 20, 30);
}
// Click to cycle
canvas.addEventListener('click', () => { testLevel = (testLevel + 1) % 5; bgCache = null; initAmbientParticles(testLevel); });
initAmbientParticles(0);
```

Take screenshots of each level background, verify they look atmospheric and distinct.

- [ ] **Step 3: Remove test code, commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: procedural backgrounds for all 5 levels with ambient particles"
```

---

### Task 8: Title Screen and Level Intro

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement UI rendering for title screen and level intro**

Build the DOM-based UI screens and wire up the state machine transitions. Implement the `update()` and `render()` functions that dispatch based on `state.gamePhase`.

The title screen shows: game title, animated angel, campaign progress dots, NEW CAMPAIGN button. The level intro shows: world name, boss silhouette, flavor text, BEGIN button.

This is the first real state machine wiring — implement `showTitleScreen()`, `showLevelIntro()`, the overlay HTML generation, and the phase transitions with fades and sound.

Connect the game loop's `update(dt)` to call `updateParticles(dt)`, `updateAmbientParticles(dt)`, `updateScreenEffects(dt)`, `updateFade(dt)`, `updateFloatTexts(dt)`.

Connect `render()` to draw the background, sprites, particles, effects, fade overlay based on `state.gamePhase`.

- [ ] **Step 2: Screenshot title screen and level 1 intro**

Take screenshots, verify: title looks polished with golden text, angel sprite visible, button is clickable. Level intro shows "LEVEL 1 — ENCHANTED FOREST" with forest background, button works.

- [ ] **Step 3: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: title screen, level intro, state machine wiring"
```

---

### Task 9: Training Phase

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement training phase**

Wire up the training phase with:
- Question display and answer buttons
- Timer (15s countdown with tick sounds)
- Correct/wrong answer handling with sound and visual feedback
- Correct answer reveal on wrong (highlight green for 1.5s)
- Power bar filling
- Streak tracking and streak bonus at 3
- Angel evolution at thresholds (with sfxPowerUp, sprite swap, flash)
- Score calculation (speed-based points + streak multiplier)
- Training complete transition ("READY FOR BATTLE" + dramatic darkening)

Handle all the HUD elements: power bar, streak counter, stage name, timer ring, score display.

- [ ] **Step 2: Playtest training phase — play through Level 1 training**

Open in Chrome, click through title → level 1 intro → training. Answer questions, verify:
- Questions show x2 table
- Timer counts down with tick sounds
- Correct: chime, flash, power bar fills, score adds
- Wrong: buzz, correct answer shown in green, streak resets
- Streak of 3: arpeggio, sparkle
- Evolution transitions work (sprite changes at thresholds)
- "READY FOR BATTLE" appears after all 6 questions

Take screenshots at: start of training, mid-training with power bar, evolution moment, "READY FOR BATTLE".

- [ ] **Step 3: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: training phase with questions, timer, evolution, scoring"
```

---

### Task 10: Battle Phase (Core Combat)

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement battle phase**

Wire up the battle phase with:
- Battle transition (fade, sfxBattleStart, boss entrance)
- Battle question display with shorter timer per level
- Angel attacks on correct answer (Light Beam / Holy Slash / Divine Strike based on speed)
- Boss attacks on wrong answer (unique animation per boss)
- HP bar management for both sides
- Hit reactions (flash, knockback)
- Screen shake and screen flash on attacks
- Floating text for attack names and damage
- Win condition (boss HP <= 0, victory animation, sfxVictory)
- Lose condition (angel HP <= 0, defeat screen, sfxDefeat, retry)
- Question priority queue (wrong answers from training first)
- Correct answer reveal on wrong answer (green highlight 1.5s, then boss attacks)

- [ ] **Step 2: Implement angel attack animations**

Light Beam: horizontal beam of particles from angel to boss.
Holy Slash: golden crescent arc bezier.
Divine Strike: angel lifts, zoom, golden column from above, heavy shake.

Use the animation queue system — each animation is a timed sequence of effects that plays out before the next turn.

- [ ] **Step 3: Implement boss attack animations**

Each boss has a unique attack. Implement all 5:
- Shadow Wolf: dark streak + jaw snap particles
- Frost Witch: ice shard triangles flying L
- Kraken: tentacle sweep bezier
- Devil Queen: fireball arc + ember burst
- Violet Avenger: screen dim + purple beam

- [ ] **Step 4: Playtest a full Level 1 (training + battle)**

Play through the entire Level 1. Verify:
- Battle transition works (fade, horn, boss appears)
- Questions appear with timer
- Fast correct: Divine Strike (big animation, heavy shake)
- Slow correct: Light Beam (simple beam)
- Wrong answer: correct shown, then boss attacks
- HP bars update smoothly
- Win: boss dissolves, victory screen shows stats
- Retry on lose works (goes back to level intro)

Take screenshots of: battle start, angel attacking, boss attacking, victory screen, defeat screen.

- [ ] **Step 5: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: battle phase with combat, animations, HP, win/lose"
```

---

### Task 11: Super Powers

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement super power selection and effects**

Every 3rd battle question: show 3 power icons for player to choose (Wings of Fire, Holy Lightning, Divine Shield). After selection, show the question.

- Wings of Fire: double next attack damage, flame particles on angel wings
- Holy Lightning: 1 hit to boss + 1 heal to angel, lightning visual
- Divine Shield: golden dome around angel, persists until wrong answer or end of battle

Handle edge cases:
- Wings of Fire wasted on wrong answer
- Holy Lightning triggers on correct alongside normal attack
- Divine Shield activates immediately, consumed on next boss attack

- [ ] **Step 2: Playtest super powers in battle**

Play a battle and trigger all 3 super powers. Verify visuals and effects work correctly.

- [ ] **Step 3: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: super power selection and effects (fire, lightning, shield)"
```

---

### Task 12: Victory, Defeat, and Campaign Flow

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Implement victory screen**

Stats panel: questions answered, accuracy %, fastest answer, best streak. Score breakdown with level multiplier. "NEXT LEVEL" button that advances to next level intro. World-specific celebration particles.

- [ ] **Step 2: Implement defeat screen**

Encouraging message with progress: "You dealt X% damage!" Progress bar. "RETRY" button (resets level state, keeps totalScore). Flavor text encouraging retry.

- [ ] **Step 3: Implement campaign complete screen**

After Level 5 victory: grand celebration, all 5 bosses shown defeated, total stats, credits with sprite attribution, "PLAY AGAIN" button.

- [ ] **Step 4: Wire up full campaign flow**

Verify the full loop: title → L1 intro → L1 training → L1 battle → L1 victory → L2 intro → ... → L5 victory → campaign complete → title.

Also verify: defeat → retry → same level intro. Score accumulates across levels. Campaign progress dots update on title screen.

- [ ] **Step 5: Playtest full campaign flow (at least levels 1-3)**

Play through levels 1, 2, and 3 to verify progression works. Take screenshots of each victory screen and the transition to the next level.

- [ ] **Step 6: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: victory, defeat, campaign complete screens, full campaign flow"
```

---

### Task 13: Visual Polish

**Files:**
- Modify: `angel-vs-devil.html`

- [ ] **Step 1: Seraphim aura effect**

For angel stage 4: add animated golden aura (radial gradient behind sprite), sparkle particles around the angel, light rays from above. Use the `character-art-preview.html` Seraphim post-effect code as reference.

- [ ] **Step 2: Frost Witch ice tint**

Apply blue-ice tint to witch sprite using `source-atop` compositing with `rgba(100,180,255,0.3)`. Add persistent frost particle effects around her during battle.

- [ ] **Step 3: Boss entrance animations**

Implement unique entrance for each boss when battle begins:
- Shadow Wolf: dark shape lunges in from right
- Frost Witch: floats in from above with ice sparkle trail
- Kraken: tentacles rise from bottom edge, then head appears
- Devil Queen: fire eruption, she materializes
- Violet Avenger: dark energy coalesces

Each is a 1-2 second animation that plays after sfxBattleStart before combat begins.

- [ ] **Step 4: Evolution transitions polish**

Add smoother evolution transitions: old sprite fades out (0.3s), white flash, new sprite fades in (0.3s), stage name text scales up then fades, particle burst in the evolution's color theme.

- [ ] **Step 5: Idle bobbing animation**

Both angel and boss should bob vertically 1-2px using a sine wave on each frame during idle (waiting for player input).

- [ ] **Step 6: Screenshot all 5 levels to verify visual quality**

Take screenshots of each level's training and battle phases. Verify backgrounds, sprites, particles, effects all look cohesive.

- [ ] **Step 7: Commit**

```bash
git add angel-vs-devil.html
git commit -m "feat: visual polish — aura, ice tint, boss entrances, evolution FX, idle bob"
```

---

### Task 14: Full Playtest and Bug Fix Pass

**Files:**
- Modify: `angel-vs-devil.html`
- Create: `docs/playtest-log.md`

- [ ] **Step 1: Play through the entire game as an 8-year-old**

Play through all 5 levels from start to finish. Approach it as an 8-year-old would:
- Click the shiny button
- Try to answer quickly (sometimes correctly, sometimes not)
- Get excited about power-ups and attacks
- Intentionally get some wrong to test wrong-answer flow
- Intentionally let timer run out at least once
- Trigger all 3 super powers
- Lose at least one boss fight to test retry
- Win the full campaign

For each level, take a screenshot of: training start, mid-training, battle start, an angel attack, a boss attack, and the result screen.

- [ ] **Step 2: Write detailed playtest log**

Create `docs/playtest-log.md` with findings organized by category:

```markdown
# Playtest Log — Angel vs Devil

## Date: 2026-04-03

### Visual Issues
- [ ] Issue description (screenshot: /tmp/playtest-X.png)

### Gameplay Issues
- [ ] Issue description

### Sound Issues
- [ ] Issue description

### UX/Flow Issues
- [ ] Issue description

### Things That Feel Great
- (What's working well)

### Things That Need Work
- (What an 8-year-old would find confusing, boring, or frustrating)
```

- [ ] **Step 3: Fix all issues found**

Work through each issue in the playtest log. For each fix:
1. Make the code change
2. Verify with a screenshot
3. Check the item off in the log

- [ ] **Step 4: Second playtest pass**

Play through levels 1, 3, and 5 again after fixes. Verify all issues are resolved. Take fresh screenshots.

- [ ] **Step 5: Final commit**

```bash
git add angel-vs-devil.html docs/playtest-log.md
git commit -m "fix: playtest fixes — visual, gameplay, UX improvements"
```

---

### Task 15: Final Verification

**Files:**
- Modify: `angel-vs-devil.html` (if needed)

- [ ] **Step 1: Verify responsive scaling**

Test in Chrome at different viewport sizes:
- Full desktop (1920x1080)
- Laptop (1366x768)
- Tablet landscape (1024x768)
- Phone landscape (667x375)

Use Chrome DevTools device toolbar to test each. Screenshot each. Verify:
- Canvas scales correctly maintaining 16:10
- Answer buttons are large enough to tap (min 48px)
- HUD text is readable
- No overflow or cutoff

- [ ] **Step 2: Verify offline functionality**

Disconnect network, refresh page. Verify game loads and plays fully offline.

- [ ] **Step 3: Verify attribution**

Check the campaign complete screen shows proper sprite attribution:
- Jordan Irwin (AntumDeluge)
- Svetlana Kushnariova (Cabbit)
- Stephen Challener (Redshrike)
- William Thompson
- Lanea Zimmerman
- Diligent Dodo (soniccuz)

With license info (CC-BY 3.0/4.0, OGA-BY 3.0, CC-BY-SA 3.0).

- [ ] **Step 4: Final commit if any changes**

```bash
git add angel-vs-devil.html
git commit -m "fix: final responsive, offline, and attribution verification"
```
