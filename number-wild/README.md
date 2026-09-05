# NumberWild

A single-file, offline number-learning game for iPad (or any browser), built as a sibling to WordWild. Same Pokémon-style "catch and level up" adventure — this time the Wildlings are squares, and you catch numbers and math facts. No app store, no build tools — open the HTML file and play.

## What it is

Your kid explores a map of **regions**, each split into **zones**. A square critter appears, a question is spoken, and your kid taps the matching answer among a few choices to "catch" it. Catch an item 5 times and its critter maxes out (wings → spike → crown) and retires from the rotation. Clear enough catches in a zone and its **Gym Leader** appears — a 3-in-a-row streak challenge. Clear every gym in a region and the **Region Master** unlocks — a 5-in-a-row challenge across every item in that region. **Operator Ridge stays locked until you beat Pebble Master** — clearing gyms alone is not enough. Beating a region's Master is the only way to unlock the next region. A locked teaser tile at the bottom of the map shows what comes next.

A **Dex** screen (tabbed by region) shows every critter discovered so far, with star ratings, and "???" placeholders for anything not yet caught.

## How to use it

1. Open `numberwild.html` in Safari on the iPad.
2. Tap Share → **Add to Home Screen** so it runs full-screen like a real app.
3. Progress saves automatically to that browser/device — no account, no login, no data leaves the device.

## How it's built

Everything lives in one HTML file: markup, CSS, and JavaScript, no external dependencies except the browser's built-in `speechSynthesis` (for the 🔊 button) and a small `window.storage` API (for saving progress). There's no build step — editing the file and reloading is the whole workflow.

### The core data structure: `REGIONS`

Everything the game knows about content lives in one array near the top of the `<script>` tag. Each zone has a `type` so the battle screen can ask different kinds of questions:

```js
const REGIONS = [
  {
    name: "Pebble Plains",
    subtitle: "Sight Numbers",
    masterTitle: "Pebble Master",
    zones: [
      { name: "Counting Glade", icon: "🟩", type: "identify", items: rangeItems(0,10), catchesForBoss: 8 },
      // ...
    ]
  },
];
```

That's the entire content model. Catching, gyms, the master trial, and the Dex are generic over `items`. Only `makeQuestion()` / `distractorsFor()` / `speakQuestion()` branch on `type`.

### Shipped regions

**Pebble Plains — Sight Numbers** (identify spoken numbers, hide the glyph)

| Zone | Range | Items | Gym after |
| --- | --- | --- | --- |
| Counting Glade | 0–10 | 11 | 8 catches |
| Teen Trail | 11–20 | 10 | 8 |
| Fifty Fields | 21–50 | 30 | 12 |
| Eighty Hills | 51–80 | 30 | 12 |
| Century Ridge | 81–100 | 20 | 10 |
| Pebble Master | 0–100 mix | all 101 | 5-streak |

Ranges are exclusive at the shared decade so each number 0–100 is one Dex entry (catching `10` in Glade does not also fill Teen Trail).

**Operator Ridge — Facts & Place Value**

| Zone | Type | Pool | Gym after |
| --- | --- | --- | --- |
| Sum Springs | `add` | all `a+b` with a,b in 0–9 (100) | 12 |
| Difference Dell | `sub` | all `a-b` with a ≥ b, a,b in 0–9 (55) | 12 |
| Ones Outpost | `ones` | 10+d and 20+d for d=0–9, plus 7 and 100 (22) | 8 |
| Tens Tower | `tens` | t×10 and t×10+7 for t=1–9 (18) | 8 |
| Operator Master | mix | all of the above | 5-streak |

Math facts use unique keys (`3+5`, `ones:47`, `tens:47`) so they never collide with Realm 1's `"47"`.

### Question types

- **`identify`** — speak the number, hide the numeral, 4 number choices. Distractors prefer near-misses (`n±1`, `n±2`, `n±10`, digit reverse like 17/71).
- **`add` / `sub`** — show the equation, speak “three plus five” / “nine minus four”, choices are numeric answers.
- **`ones` / `tens`** — show the full number, ask which digit is in that place, choices are 4 digits.

### How critters are generated (no art files needed)

`creatureSVG(id, level)` hashes the item key into a number, then picks a color palette, ear shape, and eye style. Bodies are **rounded squares** with blocky ears and feet (WordWild uses circles). The same item always produces the same-looking critter. Level overlays: wings at 3, spike at 4, crown at 5.

### Difficulty knobs

```js
const MAX_LEVEL = 5;        // catches needed to max out an item
const GYM_STREAK = 3;       // correct-in-a-row to beat a gym
const MASTER_STREAK = 5;    // correct-in-a-row to beat a Region Master
```

Per-zone `catchesForBoss` scales with pool size so a 30-number field isn't gym-ready after eight taps.

### Adding a new region

Append an object to `REGIONS` with `name`, `subtitle`, `masterTitle`, and `zones`. Each zone needs `name`, `icon`, `type`, `items`, and optionally `catchesForBoss`. Reuse an existing `type` (`identify`, `add`, `sub`, `ones`, `tens`) and no battle code has to change. Item keys should stay unique game-wide — the critter look is generated from the key itself.

To add a new question type, add a branch in `makeQuestion()` and `distractorsFor()`. Catch / gym / master / Dex stay unchanged.

### Save data

Progress is stored under the key `numberwild-save` via `window.storage`, as one JSON blob:

```json
{
  "items": { "7": 3, "3+5": 1, "9-4": 0, "ones:47": 2, "tens:47": 0 },
  "regionUnlocked": 1,
  "zoneUnlockedInRegion": { "0": 4, "1": 2 },
  "bossCleared": { "0-0": true, "0-1": true },
  "masterCleared": { "0": true }
}
```

Wiping progress means clearing this key. There is no migration logic — renamed item keys start fresh at 0.
