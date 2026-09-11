# NoteWild

A single-file, offline sheet-music reading game for iPad (or any browser), built as a sibling to WordWild, NumberWild, and PianoWild. Same Pokémon-style "catch and level up" adventure — the Wildlings are note-head critters, but this time the only skill is **reading staff notes**: a note appears on a treble or bass staff, and your kid taps the matching letter. No piano keys, no app store, no build tools — open the HTML file and play.

## What it is

Your kid explores a map of **regions**, each split into **zones**. A critter appears, a staff note is shown, and your kid taps the matching letter to "catch" it. Catch an item 5 times and its critter maxes out (wings → spike → crown) and retires from the rotation. Clear enough catches in a zone and its **Gym Leader** appears — a 3-in-a-row streak. Clear every gym in a region and the **Region Master** unlocks — a 5-in-a-row challenge across every note in that region's scale.

Beating the Treble and Bass masters (both cleared together) unlocks the third region, **Clefs Crossing**, where each question draws a note in a random clef so your kid has to read whichever staff is shown. A locked teaser tile on the map shows what comes next until both masters fall.

A **Dex** screen (tabbed by region) shows every critter discovered so far, with star ratings, and "???" placeholders for anything not yet caught.

## How to use it

1. Open `notewild.html` in Safari on the iPad.
2. Tap Share → **Add to Home Screen** so it runs full-screen like a real app.
3. Progress saves automatically to that browser/device — no account, no login, no data leaves the device.

## How it's built

Everything lives in one HTML file: markup, CSS, and JavaScript, no external dependencies except the browser's built-in `speechSynthesis` (for voicing question prompts), Web Audio (for the note tones), and a small `window.storage` API (for saving progress). There's no build step — editing the file and reloading is the whole workflow.

### The core data structure: `REGIONS`

Everything the game knows about content lives in one array near the top of the `<script>` tag. Each region carries a `clef` (`treble`, `bass`, or `mixed`) that decides which staff is drawn and which pitch table is used when the speaker plays the tone:

```js
const TREBLE_LETTERS = ['C','D','E','F','G','A','B'];
const BASS_LETTERS   = ['G','A','B','C','D','E','F'];
const FREQ = {
  treble: {C:261.63, D:293.66, E:329.63, F:349.23, G:392.00, A:440.00, B:493.88},
  bass:   {G:98.00,  A:110.00, B:123.47, C:130.81, D:146.83, E:164.81, F:174.61},
};

const REGIONS = [
  {
    name: "Treble Meadows", subtitle: "Treble Clef", masterTitle: "Treble Master", clef: "treble",
    zones: clusterZones('treb', 'name-note', TREBLE_LETTERS,
      ['EFG Brood','FGA Fern','GAB Rise','Full Treble'],
      ['🎵','🪶','🎶','✨'])
  },
  // ... Bass Falls (clef: "bass") and Clefs Crossing (clef: "mixed")
];
```

That's the entire content model. Catching, gyms, the master trial, and the Dex are generic over `items`. Only `makeQuestion()` / `buildAnswers()` / the staff renderer branch on the note's `clef`.

### Shipped regions

Two clefs, one octave of notes each. The treble pool runs **C–B** (C on the bottom ledger line up to B above the staff); the bass pool runs **G–F** (G on the second line down to F below the third line). Every region uses the same zone shape: three sliding 3-note windows, then the full-scale review zone.

| Region | Clef | Window 1 → Full zone |
| --- | --- | --- |
| Treble Meadows | treble (C–B) | CDE · EFG · GAB → full C–B |
| Bass Falls | bass (G–F) | GAB · BCD · DEF → full G–F |
| Clefs Crossing | mixed | same 3 windows, clef chosen per question |

`catchesForBoss` is 8 on each 3-note zone and 20 on the full-scale review, so the mixed zone can't unlock the gym on a lucky streak.

### The one question type: `name-note`

Show a staff note (no letter). Speak the prompt. Choices are letter buttons drawn from the region's scale. Correct answers play the pitch; the speaker never *names* the note, so the kid really has to read the position.

- **Treble** staff: treble clef glyph + C–B pool.
- **Bass** staff: bass (F) clef glyph + G–F pool.
- **Mixed** region: `resolveClef()` flips a coin per question — 50% treble, 50% bass — and the staff + answer pool follow whichever clef landed.

### How staff notes are drawn (no art files needed)

`staffSVG(letter, opts)` hashes the note into a fixed y-position on the five-line staff (with ledger lines where a note sits outside it) and draws either the treble or bass clef glyph from a small lookup table of staff positions. The same note in a given clef always renders in the same spot.

### How critters are generated (no art files needed)

`creatureSVG(id, level)` hashes the item key into a number, then picks a color palette and flag shape. Bodies are **oval note-heads** with a stem (the same family WordWild/NumberWild/PianoWild use for the wildlings). The same item always produces the same-looking critter. Level overlays: wings at 3, spike at 4, crown at 5.

### Unlock gate

Treble Meadows and Bass Falls are both open on first launch. Clefs Crossing unlocks the moment **both** of the other regions' masters are cleared — no order required. The rule lives in `highestUnlockedRegion()` and is recomputed from `masterCleared` every render, so there's no stored bit to stay in sync.

### Difficulty knobs

```js
const MAX_LEVEL = 5;        // catches needed to max out an item
const GYM_STREAK = 3;       // correct-in-a-row to beat a gym
const MASTER_STREAK = 5;    // correct-in-a-row to beat a Region Master
```

Per-zone `catchesForBoss` is 8 on the three-note windows and 20 on the full-scale review.

### Adding a new region

Append an object to `REGIONS` with `name`, `subtitle`, `masterTitle`, `clef`, and `zones`. Set `clef` to `treble`, `bass`, or `mixed`, and point the zones at the right letter pool — no battle code has to change. Item keys should stay unique game-wide — the critter look is generated from the key itself.

To add a new question type, add a branch in `makeQuestion()` and `buildAnswers()`. Catch / gym / master / Dex stay unchanged.

### Save data

Progress is stored under the key `notewild-save` via `window.storage`, as one JSON blob:

```json
{
  "items": { "treb:C": 3, "bass:E": 1, "mix:B": 0 },
  "regionUnlocked": 2,
  "zoneUnlockedInRegion": { "0": 3, "1": 0 },
  "bossCleared": { "0-0": true, "1-0": true },
  "masterCleared": { "0": true, "1": true }
}
```

Wiping progress means clearing this key. There is no migration logic — renamed item keys start fresh at 0.
