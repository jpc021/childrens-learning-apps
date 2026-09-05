# WordWild

A single-file, offline sight-word learning game for iPad (or any browser), built as an original Pokémon-style "catch and level up" adventure. No app store, no build tools — open the HTML file and play.

## What it is

Your kid explores a map of **regions**, each split into 8 **zones**. Every zone is tied to a small set of sight words. A critter appears, speaks its word out loud, and your kid taps the matching word among a few choices to "catch" it. Catch a word 5 times and its critter maxes out (wings → spike → crown) and retires from the rotation. Clear enough catches in a zone and its **Gym Battle** starts — get 3 correct in a row to beat the gym; 3 misses and you return to the map to retry. Words are spoken with **David** or **Zira** when those voices are installed (Safari falls back to Alex/Daniel/Fred and Samantha/Victoria/Karen), a random gender per question. Clear every gym in a region and the **Region Master** unlocks — a 5-in-a-row challenge across every word in that region. Beating the Master unlocks the next region.

A **Dex** screen (tabbed by region) shows every critter discovered so far, with star ratings, and "???" placeholders for anything not yet caught.

## How to use it

1. Open `wordwild.html` in Safari on the iPad.
2. Tap Share → **Add to Home Screen** so it runs full-screen like a real app.
3. Progress saves automatically to that browser/device — no account, no login, no data leaves the device.

## How it's built

Everything lives in one HTML file: markup, CSS, and JavaScript, no external dependencies except the browser's built-in `speechSynthesis` (for the 🔊 button) and a small `window.storage` API (for saving progress). There's no build step — editing the file and reloading is the whole workflow.

### The core data structure: `REGIONS`

Everything the game knows about content lives in one array near the top of the `<script>` tag:

```js
const REGIONS = [
  {
    name: "Meadow Realm",
    subtitle: "Sight Words I",
    masterTitle: "Meadow Master",
    zones: [
      { name: "Sunny Meadow", icon: "🌼", words: ["a","and","big","go","up"] },
      // ... 7 more zones
    ]
  },
  // ... more regions
];
```

That's the entire content model. Everything else — the map screen, the battles, the Dex, leveling, unlocking — is generic code that reads from this array. Adding content never means touching the game logic.

### Adding a new region

To add a third region, append a new object to `REGIONS`:

```js
{
  name: "Coral Reef",
  subtitle: "Sight Words III",
  masterTitle: "Reef Master",
  zones: [
    { name: "Tide Pools", icon: "🐚", words: ["about","after","again","an","any"] },
    // ... up to 8 zones total, ~5 words each works well
  ]
}
```

That's it — the map's region switcher, the Dex tabs, the gym/master unlock chain, and the save system all pick it up automatically. No other code needs to change as long as:

- Each zone has a unique `name`, an `icon` (any emoji), and a `words` array.
- Words across the whole game should generally be unique — the critter's look is generated from the word text itself (see below), so reusing a word across regions would show the same critter twice.

### How critters are generated (no art files needed)

`creatureSVG(word, level)` hashes the word's letters into a number, then uses that number to deterministically pick a color palette, ear shape, and eye style from small fixed lists. This means:

- The same word **always** produces the same-looking critter, every session, forever.
- No image assets, no naming files, no drawing — new words automatically get a new-looking critter for free.
- `level` (0–5, tracked per word) layers on visual upgrades: wings at level 3, a spike at level 4, a crown at level 5 (max).

If you want a genuinely different art style for a new region (e.g., "phonics region" critters look more like sound-wave creatures), you'd branch inside `creatureSVG` on which region the word belongs to — that's the one place visuals are generated.

### Difficulty knobs

Near the top of the script:

```js
const MAX_LEVEL = 5;            // catches needed to max out a word
const CATCHES_PER_BOSS = 8;     // cumulative zone catches before the gym unlocks
const GYM_STREAK = 3;           // correct-in-a-row to beat a gym
const GYM_MISSES = 3;           // misses in one gym visit = retry from the map
const MASTER_STREAK = 5;        // correct-in-a-row needed to beat the Region Master
```

Gyms are 3-in-a-row; a miss resets the streak, and 3 misses send you back to the map with the gym still open. Turn these knobs up or down as your kid's skill grows — e.g., raise `CATCHES_PER_BOSS` if gyms are unlocking too fast, or lower `MASTER_STREAK` if the master trial feels too long for a 5-year-old.

### Extending beyond words: a future math (or phonics) region

The battle screen currently assumes every question is "match this word." To support a different question type (e.g., "tap the critter with the right answer to 3 + 2"), the cleanest approach is:

1. Add a `type` field to each zone, e.g. `type: "word"` or `type: "math"`.
2. In `nextWord()` / `buildChoices()`, branch on `zone.type`: for `"word"` zones, keep the current word-matching flow; for `"math"` zones, generate a question string (e.g. `"3 + 2"`) and compute the correct numeric answer instead of pulling from a fixed word list.
3. The catching, leveling, gym, master, and Dex systems don't care what the "word" *is* — they just track a catch-count per item and render it with `creatureSVG`. A math fact like `"3+2"` can be treated exactly like a word for storage and leveling purposes; it just needs its own answer-generation logic instead of a hardcoded list.

This keeps one engine (map → zones → gym → master → Dex) serving multiple content types as the curriculum grows — sight words now, phonics blends or simple addition later — without rewriting the progression system each time.

### Save data

Progress is stored under the key `wordwild-save` via `window.storage`, as one JSON blob:

```json
{
  "words": { "a": 3, "and": 5, "big": 0, ... },
  "regionUnlocked": 1,
  "zoneUnlockedInRegion": { "0": 7, "1": 2 },
  "bossCleared": { "0-0": true, "0-1": true, ... },
  "masterCleared": { "0": true }
}
```

Wiping progress (e.g., to start over, or after big content changes) just means clearing this key — there's no migration logic, so if you rename/reorder words or zones after your kid has already played, old saved catch-counts for renamed words will simply start fresh at 0 rather than crashing anything.
