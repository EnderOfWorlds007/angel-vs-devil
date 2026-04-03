# Angel vs Devil — Multiplication Game

## What to Build

Build a single-file HTML game (`angel-vs-devil.html`) based on the spec in `angel-vs-devil-game-spec.md`. Read that file thoroughly before starting — it contains everything: campaign structure, game flow, combat visuals, boss scaling, sound recipes, and technical requirements.

## Project Files

```
angel-vs-devil-game-spec.md   — THE SPEC. Read this first. It's the source of truth.
sprites_b64.json               — Base64-encoded sprite data (keys: fallen1, fallen2, dark, angel, archangel, devil, violet)
character-art-preview.html     — Interactive preview showing how sprites are loaded, frames extracted, and effects applied
question-ui-mockup.html        — Interactive mockup of the question/answer UI for both training and battle phases
sound-effects-preview.html     — All 14 procedural sound effects with Web Audio API code you can copy
```

## Sprite Mapping

The sprites in `sprites_b64.json` map to game characters as follows:

| JSON Key | Sprite | Used For |
|---|---|---|
| `fallen2` | angel-f-002 (cream robes) | Stage 0 (reduced opacity) and Stage 1 (full opacity) |
| `angel` | angel-f-001 (blue robes) | Stage 2 — Guardian |
| `archangel` | angel-m-001 (golden armor) | Stage 3 — Archangel, and Stage 4 — Seraphim (with golden glow effects) |
| `devil` | devil_queen (SWEN layout) | Level 4 boss — Devil Queen |
| `violet` | violet_avenger (SWEN layout) | Level 5 boss — Violet Avenger |
| `fallen1`, `dark` | Not used in the game | Available but not needed |

### Sprite Sheet Format

All sprites are 48×64 pixel sprite sheets arranged in a 3-column × 4-row grid:
- Each frame is 16×16 pixels
- Rows = directions (South, West, East, North for SWEN sheets)
- Columns = animation frames (3 per direction)
- Use `drawImage(img, col*16, row*16, 16, 16, dx, dy, scaledW, scaledH)` to extract frames
- Apply `image-rendering: pixelated` on the canvas for crisp scaling

See `character-art-preview.html` for working extraction code, including the Seraphim golden glow effect (uses `source-atop` compositing, `lighter` blend, radial gradient aura, and particle overlay).

## Key Implementation Details

### Questions
- Each level uses ONE multiplication table (Level 1 = ×2, Level 2 = ×3, ... Level 5 = ×6)
- Questions are `N × (1-10)` where N is the level's table number
- Track wrong answers during training — the boss fight prioritises those questions first

### Combat
- All spell-based (no physical contact). Particle effects, beams, screen shakes.
- Angel attack tier depends on answer speed: Light Beam (<8s), Holy Slash (<5s), Divine Strike (<3s)
- Each boss has a unique themed attack animation (see spec for details)

### Sound Effects
- All procedural via Web Audio API — no audio files
- `sound-effects-preview.html` has working implementations of all 14 sounds
- Copy the oscillator/gain/filter patterns directly

### Bosses for Levels 1-3
- Shadow Wolf, Frost Witch, and Kraken Lord have no sprite files
- Draw them programmatically on canvas (simple but characterful silhouettes/shapes, 48×64 to match the pixel art style)

## Output

One file: `angel-vs-devil.html`. No dependencies, no CDN imports, works offline. Everything embedded.

## Preferences

- Always verify visual work (HTML, games, charts, etc.) by screenshotting it yourself before presenting it as ready. Never say "it works" without checking first.
