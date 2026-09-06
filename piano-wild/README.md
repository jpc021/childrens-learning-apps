# PianoWild

A single-file, offline piano-learning game for iPad (or any browser), built as a sibling to WordWild and NumberWild. Same Pokémon-style "catch and level up" adventure — this time the Wildlings are note-head critters, and you catch keys and treble-clef notes. No app store, no build tools — open the HTML file and play.

## What it is

Your kid explores a map of **regions**, each split into **zones**. A critter appears, a letter or staff note is shown, and your kid taps the matching piano key or note to "catch" it. Catch an item 5 times and its critter maxes out (wings → spike → crown) and retires from the rotation. Clear enough catches in a zone and its **Gym Leader** appears — a 3-in-a-row streak challenge. Clear every gym in a region and the **Region Master** unlocks — a 5-in-a-row challenge across every note in that region. Beating a region's Master is the only way to unlock the next region. A locked teaser tile at the bottom of the map shows what comes next.

A **Dex** screen (tabbed by region) shows every critter discovered so far, with star ratings, and "???" placeholders for anything not yet caught.

## How to use it

1. Open `pianowild.html` in Safari on the iPad.
2. Tap Share → **Add to Home Screen** so it runs full-screen like a real app.
3. Progress saves automatically to that browser/device — no account, no login, no data leaves the device.

## How it's built

Everything lives in one HTML file: markup, CSS, and JavaScript, no external dependencies except the browser's built-in `speechSynthesis` (for voicing letter names), Web Audio (for C4–B4 piano-ish tones), and a small `window.storage` API (for saving progress). There's no build step — editing the file and reloading is the whole workflow.

### The core data structure: `REGIONS`

Everything the game knows about content lives in one array near the top of the `<script>` tag. Each zone has a `type` so the battle screen can ask different kinds of questions:

```js
const REGIONS = [
  {
    name: "Ivory Meadow",
    subtitle: "Find the Key",
    masterTitle: "Ivory Master",
    zones: [
      { name: "CDE Grove", icon: "🎹", type: "find-key", items: noteItems("key", ["C","D","E"]), catchesForBoss: 8 },
      // ...
    ]
  },
];
```

That's the entire content model. Catching, gyms, the master trial, and the Dex are generic over `items`. Only `makeQuestion()` / `buildAnswers()` / `announceQuestion()` branch on `type`.

### Shipped regions

The keyboard is one octave of white keys, **middle C4 through B4**. Black keys are drawn for a real-piano look but are not tappable yet.

Every region uses the same zone pools: C D E, then E F G, then G A B, then the full C–B mix. Item keys stay unique per skill so catching C on the keyboard does not max out reading C on the staff.

**Ivory Meadow — Find the Key** (`find-key`)

Show a letter, speak it, play the pitch. Tap that key on the C–B keyboard.

| Zone | Notes | Gym after |
| --- | --- | --- |
| CDE Grove | C D E | 8 catches |
| EFG Brook | E F G | 8 |
| GAB Hill | G A B | 8 |
| Ivory Stretch | C–B | 20 |
| Ivory Master | C–B mix | 5-streak |

**Staff Springs — Name the Note** (`name-note`)

Show a treble-clef note (no letter). Choices are letter buttons. The speaker plays the pitch only.

**Melody Ridge — Play the Note** (`staff-to-key`)

Show a staff note. Tap the matching piano key.

**Echo Cliffs — Match the Staff** (`key-to-staff`)

Highlight a piano key and play its pitch. Choices are mini staff cards.

Overlapping E and G inside a region share one Dex entry. Zone 4's higher `catchesForBoss` keeps mixed practice from unlocking the gym immediately.

### Question types

- **`find-key`** — big letter + spoken name + pitch; answers are piano keys.
- **`name-note`** — treble staff; answers are letter buttons from the zone pool.
- **`staff-to-key`** — treble staff; answers are piano keys.
- **`key-to-staff`** — highlighted key; answers are staff SVGs.

The full C–B keyboard is always shown for key answers, even in the C–D–E zone, so kids learn *where* C sits.

### How critters are generated (no art files needed)

`creatureSVG(id, level)` hashes the item key into a number, then picks a color palette and flag shape. Bodies are **oval note-heads** with a stem (WordWild uses circles, NumberWild uses squares). The same item always produces the same-looking critter. Level overlays: wings at 3, spike at 4, crown at 5.

### Difficulty knobs

```js
const MAX_LEVEL = 5;        // catches needed to max out an item
const GYM_STREAK = 3;       // correct-in-a-row to beat a gym
const MASTER_STREAK = 5;    // correct-in-a-row to beat a Region Master
```

Per-zone `catchesForBoss` is 8 on the three-note zones and 20 on the full-octave review.

### Adding a new region

Append an object to `REGIONS` with `name`, `subtitle`, `masterTitle`, and `zones`. Reuse an existing `type` (`find-key`, `name-note`, `staff-to-key`, `key-to-staff`) and no battle code has to change. Item keys should stay unique game-wide — the critter look is generated from the key itself.

To add a new question type, add a branch in `makeQuestion()` and `buildAnswers()`. Catch / gym / master / Dex stay unchanged.

### Save data

Progress is stored under the key `pianowild-save` via `window.storage`, as one JSON blob:

```json
{
  "items": { "key:C": 3, "staff:E": 1, "play:G": 0, "read:B": 2 },
  "regionUnlocked": 1,
  "zoneUnlockedInRegion": { "0": 3, "1": 0 },
  "bossCleared": { "0-0": true, "0-1": true },
  "masterCleared": { "0": true }
}
```

Wiping progress means clearing this key. There is no migration logic — renamed item keys start fresh at 0.
