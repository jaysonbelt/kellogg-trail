# The Kellogg Trail — Project Handoff

## What This Is
An Oregon Trail–style browser game based on the true origin story of corn flakes at the Battle Creek Sanitarium (1894). The player navigates Dr. John Harvey Kellogg's insane health regime, survives yogurt enemas, witnesses C.W. Post steal the cereal concept, and is present when Will Kellogg discovers corn flakes at midnight. Everything in the game actually happened.

## Current Status
- **Single file:** `kellogg-trail.html` (~1,550 lines, zero dependencies)
- **Playable end-to-end:** 8 scenes, title screen, death screen, victory screen
- **No bugs known** — syntax validated, all runtime patterns checked

---

## File Structure (everything is in one HTML file)

```
kellogg-trail.html
├── <style>          — minimal CSS (canvas centering, pixel rendering)
└── <script>
    ├── CANVAS SETUP          — 640×400, scales to window
    ├── DRAWING PRIMITIVES    — box(), hln(), vln(), dbox(), T(), TC(), TR(), wrap()
    ├── CGA PALETTE           — C.BK, C.GR, C.YL, C.CY, C.RD, C.SKY, C.SKIN, etc.
    ├── PIXEL CHARACTERS      — 2×2 pixel grid sprites, all drawn on main canvas
    ├── SCENE RENDERERS       — 8 scrolling backgrounds
    ├── NPC PLACEMENT         — placeNPCs() puts right characters in each scene
    ├── OUTCOME SPRITES       — good/bad/neutral/victory/death animated sprites
    ├── HUD                   — top bar with Sanity, Bowel, Enemas, Kellogg approval, Day
    ├── TEXT BOX              — bottom-half DOS-style event panel
    ├── GAME DATA             — EVENTS[] array (8 events, 3 choices each)
    ├── INPUT                 — mouse, touch, keyboard (1/2/3, Space, Enter)
    ├── GAME LOGIC            — state machine + stat tracking
    └── MAIN LOOP             — requestAnimationFrame, dt-normalized
```

---

## Game Layout (Oregon Trail style)

```
┌─────────────────────────────────────────┐  ← 640px wide
│  HUD BAR  [Sanity][Bowel][Enemas][Day]  │  22px — always visible
├─────────────────────────────────────────┤
│                                         │
│   SCROLLING SCENE (canvas-drawn)        │  ~188px
│   Characters mid-ground, world scrolls  │
│                                         │
├═════════════════════════════════════════┤  GY = 210 (ground line)
│  DAY X -- EVENT TITLE       LOCATION   │
│  Narrative text (white)                 │
│  Dialogue (cyan)                        │
│  HIST: fact line (yellow)               │  ~190px
│  ─────────────────────────────────────  │
│  1. Choice one                          │
│  2. Choice two                          │
│  3. Choice three                        │
└─────────────────────────────────────────┘
```

During **outcome**, the bottom half splits:
- Left 220px: animated 8-bit sprite (bowl/tombstone/shrug)
- Right side: result text + stat changes + continue prompt

---

## Characters

| Name | Function | Scene(s) |
|------|----------|----------|
| `drawPatient()` | The player — Victorian gent in sanitarium gown | All (left side, always walking) |
| `drawKellogg()` | Dr. John Harvey Kellogg — white coat, tall hat, pointing, intense beard | approach, lobby, lecture, office |
| `drawWill()` | Will Kellogg — dark coat, stooped, clipboard, exhausted eyes | kitchen_day, kitchen_night, office |
| `drawPost()` | C.W. Post — fancy suit, gold chain, scheming eyes | sunroom |
| `drawNurse()` | Nurse — white uniform, red cross cap, carrying prop varies | hydrotherapy, lobby, kitchen_day, victory |
| `drawDoctor()` | Generic sanitarium doctor — stethoscope, spectacles | lobby, hydrotherapy, sunroom |
| `drawFord()` | Henry Ford cameo — arms crossed, disapproving | lecture |

### Nurse carrying props
```js
drawNurse(ox, oy, frame, 'tray')   // serving tray with bowl
drawNurse(ox, oy, frame, 'pot')    // yogurt pot
drawNurse(ox, oy, frame, 'enema')  // the rubber bag + tube
```

---

## Scenes

| Key | Description | Characters placed |
|-----|-------------|-------------------|
| `'approach'` | Exterior — sanitarium building, fence, trees | Kellogg, Nurse(tray) |
| `'lobby'` | Interior — paneling, grandfather clock, potted ferns | Kellogg, Doctor, Nurse(tray) |
| `'lecture'` | Lecture hall — blackboard with Kellogg's diagrams, seated patient silhouettes | Kellogg, Ford |
| `'hydrotherapy'` | Hydrotherapy wing — tiled walls, scrolling yogurt tubs with patients | Nurse(enema), Doctor, Nurse(pot) |
| `'kitchen_day'` | Kitchen daytime — brick walls, grain rollers, boiling vats, window | Will, Nurse(tray) |
| `'sunroom'` | Sunroom — large windows, light beams, wicker chairs, bowls on tables | Post, Doctor |
| `'kitchen_night'` | Kitchen at night — lamp glow, corn flakes scattered on counters | Will |
| `'office'` | Kellogg's office — dark paneling, bookshelves, desk with disputed flakes | Kellogg, Will |
| `'victory'` | Victory dawn — golden light, rising sun, corn flakes floating as leaves | Will, Nurse(tray), Kellogg (frozen/distant) |

---

## Game State Machine

```
'title' → (click/space) → 'travel'
'travel' → (progress bar fills) → 'event'
'event' → (pick choice 1/2/3) → 'outcome'
'outcome' → (continue) → 'travel' | 'death' | 'victory'
'death' → (try again) → 'title'
'victory' → (play again) → 'title'
```

### State variables
```js
let state = 'title';            // current state
let G = {                       // player stats
  sanity: 100,                  // 0 = death
  bowel: 3,                     // 0-4, affects bowel status label
  approval: 5,                  // Kellogg approval, 0-5 stars
  enemas: 0,                    // counter, no death effect, comedic
  day: 1                        // display only
};
let sceneIdx = 0;               // which event in EVENTS[] we're at
let currentEvent = null;        // active event object
let outcomeData = null;         // { type, msg, statChanges[] }
let scroll = 0;                 // world scroll X (always incrementing)
let walkF = 0;                  // animation frame counter (always incrementing)
let travelDist = 0;             // progress toward next event
let travelGoal = 100;           // randomized per travel segment
```

---

## EVENTS Array Structure

```js
{
  scene: 'approach',           // which scene renderer to use
  day: 1,                      // shown in HUD and header
  npc: 'Kellogg',              // display name in header (cosmetic)
  header: 'DAY 1 -- ARRIVAL',  // top-left of text box
  sub: 'BATTLE CREEK...',      // top-right of text box
  lines: [                     // narrative lines (empty string = blank line)
    'Line of text...',
    '"Dialogue in cyan"',
    '',
    'WILL: "Will-specific dialogue"',
  ],
  fact: 'NOTE: Historical footnote shown in yellow',
  choices: [
    {
      k: '1',                  // key label
      t: 'Choice text',        // displayed to player
      o: 'good',               // 'good' | 'neutral' | 'bad' — controls outcome sprite
      e: { sanity: 5, bowel: 1, approval: -2, enemas: 1 },  // stat deltas (all optional)
      r: 'Result text shown after choosing'
    },
    // ... 3 choices per event
  ]
}
```

### Outcome types → sprites
- `'good'` → animated corn flakes bowl with sparkles + smiling face
- `'bad'` → Oregon Trail–style tombstone with flickering candles + enema pot
- `'neutral'` → shrugging patient with question marks

---

## Stat Ranges & Effects

| Stat | Range | Death condition |
|------|-------|----------------|
| `sanity` | 0–100 | 0 = death screen |
| `bowel` | 0–4 | No death, label changes |
| `approval` | 0–5 | No death, affects star display |
| `enemas` | 0–∞ | No death, comedic counter |

---

## Known Bugs Fixed (history)

All previous bugs were in prior iterations. Current file is clean:
- No `~~v .toString()` operator precedence bug (was `(~~v).toString()`)
- No duplicate `const` in same switch case scope
- No unclosed bracket in SCENES/EVENTS array
- No null getElementById (pure canvas, one canvas element)

---

## Potential Next Steps

- **Sound** — 8-bit beeps on choice select, chiptune travel music
- **More events** — post-sanitarium epilogue (Will founds Kellogg Company, John sues)
- **Animated transitions** — screen wipe between scenes
- **Difficulty modes** — "Kellogg Easy" (more sanity), "Historical Accuracy" (harder)
- **Mobile layout** — large tap targets for choice buttons on small screens
- **Save state** — localStorage for progress (currently resets on refresh)
- **More characters** — Ella Kellogg (John's wife), other famous sanitarium patients (Sojourner Truth stayed there)
- **Mini-games** — chewing counter (click 32 times), or dodge the enema nurse

---

## Running It

Just open `kellogg-trail.html` in any browser. No server, no build step, no dependencies. Works offline. The Google Fonts `Press Start 2P` call in prior versions has been removed — current version uses `"Courier New", monospace` for the DOS aesthetic.

> Built across one Claude.ai session, March 2026. All historical facts verified. The yogurt enemas were real.
