# Angel vs Devil — Online Multiplayer Design Spec

## Overview

Cross-device PvP multiplayer using PeerJS (WebRTC). Each player gets full-screen view on their own device. Host/Client architecture — host runs game logic, client sends inputs and renders from state snapshots. Players connect via 4-character room code or QR code.

## File Structure

```
online.html         — Online multiplayer (new)
assets.js           — Shared code (existing, unchanged)
angel-vs-devil.html — Campaign (update nav: add ONLINE button)
multiplayer.html    — Same-screen PvP (update nav: rename to SAME SCREEN, add ONLINE link)
```

## PeerJS Integration

Load via CDN: `<script src="https://unpkg.com/peerjs@1/dist/peerjs.min.js">`

- Host: `new Peer('avd-XXXX')` where XXXX is a random 4-char alphanumeric code
- Client: `peer.connect('avd-XXXX')`
- Data flows over `conn.send(obj)` / `conn.on('data', callback)`
- If PeerJS cloud signaling is down, show error message

## Connection Flow

### Host Creates Game
1. Opens `online.html`, selects level (1-5) and difficulty (easy/normal/hard)
2. Picks role preference: Angel / Devil / Random
3. Clicks "CREATE GAME"
4. PeerJS creates peer with ID `avd-XXXX` (random 4-char code: A-Z, 0-9 excluding ambiguous chars I/O/0/1)
5. Screen shows: large room code "XXXX", QR code (URL: `https://enderofworlds007.github.io/angel-vs-devil/online.html?join=XXXX`), and "Waiting for opponent..."
6. QR code encodes the join URL so scanning auto-opens the page and pre-fills the code

### Client Joins
1. Opens `online.html` (or scans QR code which pre-fills the code via URL param `?join=XXXX`)
2. Enters 4-char code, clicks "JOIN GAME"
3. PeerJS connects to host
4. Host sends game config: `{ level, difficulty, hostRole, clientRole }`
5. Client receives config, both screens show: "CONNECTED! Player 1: Angel, Player 2: Devil"
6. 3-2-1 countdown, then training begins

## Host/Client Architecture

### Host Responsibilities
- Runs the complete game loop (training + battle state machine)
- Processes all game logic: HP, damage, questions, timers, power effects
- Processes client inputs as if they were local actions
- Sends state snapshots to client every 100ms (~10/sec)
- Sends event messages for sound/effect triggers

### Client Responsibilities
- Sends input messages to host on each action
- Receives state snapshots and renders from them
- Plays sounds locally based on event messages from host
- Handles local interpolation for smooth rendering between snapshots

### Messages: Client → Host
```javascript
{ type: 'shoot' }
{ type: 'answer', value: 18 }         // training or battle answer
{ type: 'power', choice: 'fire' }     // super power selection
```

**Input sequencing:** Host processes client inputs in order received. If an answer arrives after the question has already timed out on host, host ignores it (answerLocked is already true). Client may see a brief desync but the next state snapshot corrects it. This is acceptable for a kids' game — perfect sync isn't needed.

### Messages: Host → Client (every 100ms)
```javascript
{
  type: 'state',
  phase: 'battle',                     // training | battle | round-end | match-end
  round: 1,
  scores: [0, 0],
  
  host: {
    role: 'angel', hp: 12, hpMax: 15, stage: 3, powerBar: 80,
    shieldCharges: 0, knockback: 0, flash: 0, rise: 0,
  },
  client: {
    role: 'devil', hp: 10, hpMax: 15, stage: 2, powerBar: 60,
    shieldCharges: 1, knockback: 0, flash: 0, rise: 0,
  },
  
  // Only sent when there's a question for the client
  clientQuestion: { a: 3, b: 7, options: [18, 21, 24, 15] } | null,
  clientAnswerLocked: false,
  clientPowerSelect: false,            // true when client should show power picker
  
  battlefield: {
    angelX: 150, angelY: 200, bossX: 550, bossY: 200,
    bossKnockback: 0, angelKnockback: 0,
  },
}
```

### Messages: Host → Client (events, sent immediately when they happen)
```javascript
{ type: 'sfx', name: 'sfxAttackHit' }
{ type: 'sfx', name: 'sfxDivineStrike' }
{ type: 'effect', name: 'lightBeam', params: { ax, ay, bx, by } }
{ type: 'effect', name: 'divineBarrage', params: { ax, ay, bx, by } }
{ type: 'particle', burst: { x, y, count, color, speed, life } }
{ type: 'floatText', text: '-5', x: 400, y: 200, color: '#ff4400', size: 20 }
{ type: 'shake', intensity: 5 }
{ type: 'flash', color: 'rgba(255,100,0,0.5)', duration: 100 }
{ type: 'training-done', side: 'client' }
{ type: 'round-result', winner: 'host' | 'client' | 'draw' }
{ type: 'match-result', winner: 'host' | 'client', scores: [2, 1] }
```

## Rendering

### Each Player's View — Full Screen (not split)

Both players see the same layout: their character on the LEFT, opponent on the RIGHT.

**Host view (e.g., playing Angel):**
- Angel (left) vs Boss (right) — same as campaign battle layout
- Host's HP bar top-left, opponent's HP bar top-right
- Questions at bottom, full-width answer buttons
- Power selection overlay centered

**Client view (e.g., playing Devil):**
- Boss (left) vs Angel (right) — MIRRORED
- Client's HP bar top-left, opponent's HP bar top-right
- Same layout, just characters are flipped

The mirroring means: client swaps the X positions of angel and boss when rendering. "My character" is always on the left. Sprite direction: both characters use south-facing (row 0) frames — no need to flip animations. The characters face the camera, not each other (same as campaign).

### Input (same for both host and client)
- ALL letter keys + number keys = shoot (only one player per device)
- Number keys 1-4 = answer buttons (when question showing) or power selection
- Mouse/touch = click answer buttons, power cards, shoot (click on battlefield area)
- Space = shoot

### Rendering from State Snapshots (client)
- Client receives state 10x/sec
- Between snapshots, interpolate positions (lerp angelX, bossX, etc.)
- On receiving effect events, spawn the effects locally using assets.js functions
- On receiving sfx events, play the sound locally

## Training Phase

- Both players train simultaneously, full screen
- Each sees their own training (questions, power bar, evolution stages)
- Host runs both training states
- Host sends client's training state in snapshots; client renders from it
- Client sends answers via `{ type: 'answer', value: N }`
- First to finish gets +1 shield charge
- When both done: both screens show "READY FOR BATTLE!" → transition

## Battle Phase

Same mechanics as same-screen:
- Player shoots by pressing any key or tapping
- Questions every ~6s (host schedules independently for each player)
- Correct answer → power picker
- Wrong answer → opponent gets 2-damage hit
- HP: 15/15 normal, 18/18 easy, 12/12 hard
- Stage 3-4 do 2 damage per shot
- No boss fatigue (both human)
- Best of 3 rounds, roles swap each round

## Disconnection Handling

- Host monitors data channel. If no message received for >2 seconds: pause game.
- Client monitors similarly.
- Paused state: game loop stops, "RECONNECTING..." overlay shown with 10-second countdown.
- PeerJS automatically attempts reconnection.
- If reconnected within 10 seconds: host sends full state snapshot, client resumes from it. No state divergence — host is authoritative.
- If not reconnected: connected player wins by default. Show "OPPONENT DISCONNECTED — You Win!"
- On the disconnected player's screen (if they come back): "CONNECTION LOST — Game Over"

## Lobby UI

```
ANGEL vs DEVIL — ONLINE

[CREATE GAME]
  Level: [1] [2] [3] [4] [5]
  Difficulty: [EASY] [NORMAL] [HARD]
  I want to be: [ANGEL] [DEVIL] [RANDOM]
  → Shows room code + QR + "Waiting..."

[JOIN GAME]
  Enter code: [____]
  [JOIN]

[SAME SCREEN]  [CAMPAIGN]
```

If URL has `?join=XXXX` param, auto-fill the code and show the join UI.

## Navigation Updates

**angel-vs-devil.html title screen:**
- NEW CAMPAIGN
- SAME SCREEN (was MULTIPLAYER, links to multiplayer.html)
- ONLINE (links to online.html)

**multiplayer.html lobby:**
- Add "ONLINE" link to online.html
- Rename any "MULTIPLAYER" references to "SAME SCREEN"

**online.html:**
- SAME SCREEN link to multiplayer.html
- CAMPAIGN link to angel-vs-devil.html

## QR Code

Generate QR code client-side using a minimal QR library (qrcode-generator, ~4KB) or canvas-based QR drawing. Encode the URL: `https://enderofworlds007.github.io/angel-vs-devil/online.html?join=XXXX`

Decision: use a minimal embedded QR generator function (~100 lines, no dependencies). There are many public domain / MIT QR encoder implementations small enough to inline. This avoids another CDN dependency.

## Balance

Same as same-screen PvP:
- Both players: 15 HP normal, 18 easy, 12 hard
- Stage 3-4 shots do 2 damage
- Powers: fire 5 dmg, lightning 3 dmg + 2 heal, shield block 2, heal +4 HP
- Wrong answer: opponent gets 2 damage
- First to finish training: +1 shield charge

## Attribution

Same as existing: all sprite artists credited. PeerJS is MIT licensed.
