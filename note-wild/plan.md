# NoteWild

A single-file, offline **sheet-music reading** game for iPad (or any browser), built as a sibling to PianoWild, WordWild, and NumberWild. Same Pokémon-style "catch and level up" adventure. Unlike PianoWild (which leans on a playable C–B keyboard), NoteWild's whole loop is **reading noteheads on the staff** — in **treble clef**, **bass clef**, or a **mixed** mode. No piano, no app store, no build tools — open the HTML file and play.

> Status: **design draft**. This file is the build plan for `note-wild/`. The game itself does not exist yet.

## One-line goal

Teach a kid to read note names on a five-line staff in both clefs, then read the same notes in a single "mixed" mode where treble and bass notes appear together — unlocking the mixed region only after both clefs have been beaten.

## What it is

A kid explores a map of **regions**, each split into **zones**. A notehead appears on a staff (treble or bass, depending on the region) and the kid picks the matching letter from a set of choices to "catch" it. Catch a note 5 times and its critter maxes out (wings → spike → crown) and retires from the rotation. Clear enough catches in a zone and its **Gym Leader** appears — a 3-in-a-row streak. Clear every gym in a region and the **Region Master** unlocks — a 5-in-a-row streak across every note in that region. Beating the **treble** master and the **bass** master is the only way to unlock the third, **mixed** region.

A **Dex** screen (tabbed by region) shows every critter discovered, with star ratings and "???" placeholders for anything not yet caught.

## Clear difference from PianoWild

PianoWild anchors every skill to a physical keyboard (find-key, staff-to-key, key-to-staff). NoteWild has **no keyboard** — the keyboard isn't the thing to learn, *reading the staff* is. So the skill ladder is reading-only, and the clef itself is the content axis:

| Axis | PianoWild | NoteWild |
| --- | --- | --- |
| Clefs | treble only | treble **+ bass** + **mixed** |
| Answer style | tap piano key, or pick a letter | **pick a letter** (staff note shown) |
| Region count | 4 | 3 (treble / bass / mixed) |
| Unlock gate | beat each region's master in order | beat **both** master regions to open mixed |
| Note source | same C–B pool in every region | **clef-specific pools** (see ranges below) |

Because the answer is always "pick a letter from the staff note," NoteWild needs only **one question type**: `name-note`. (PianoWild has four because half of its types answer with piano keys. We drop those.) That keeps the code smaller, not bigger.

## Clef mode selection

There is no global "pick clef" setting. Choosing a clef is choosing a **region** — the map shows the two clef regions side by side and the kid (or parent) plays either one. The mixed region is the third and is locked until both are beaten. A small mode legend lives at the top of the map screen ("Treble ♭ region · Bass region · Mixed (locked)").

- **Treble region** — treble-clef staff, treble note pool.
- **Bass region** — bass-clef staff, bass note pool.
- **Mixed region** — each round randomly uses treble OR bass; note pool is the union. Locked until **both** the treble (region 0) and bass (region 1) masters are cleared (see "Region / gate change" below).

Note pools (diatonic, what a beginner sightreader knows):

| Clef | Pool | Rationale |
| --- | --- | --- |
| Treble | E F G A B C D (one octave, lines + spaces) | standard first-octave sight-reading set, same letters PianoWild uses, so the mental model carries over |
| Bass | G A B C D E F (one octave, lines + spaces) | bass clef's own octave; G2–F4 range |

Within each clef the zones sub-pool, exactly like PianoWild's `clusterZones` (three 3-note sub-pools → full pool). Letters are chosen so adjacent sub-pools overlap by one note (E-F-G / F-G-A style) to build momentum, mirroring PianoWild's CDE / EFG / GAB / full-C–B.

Concretely:

- Treble zones: **E-F-G**, **F-G-A**, **G-A-B**, then full **E–D** octave.
- Bass zones: **G-A-B**, **A-B-C**, **B-C-D**, then full **G–F** octave.

## The core data structure: `REGIONS`

Same shape as PianoWild. Three regions. Item **keys are prefixed by clef + skill** so catching G in treble never maxes out G in bass (the critter look is generated from the key, so distinct keys → distinct critters).

```js
const REGIONS = [
  {
    name: "Treble Top", subtitle: "Read Treble Clef", masterTitle: "Treble Master",
    zones: [
      { name: "Line Larks",  icon: "🪶", type: "name-note", items: noteItems("tread", ["E","F","G"]), catchesForBoss: 8 },
      { name: "Space Song",  icon: "🎵", type: "name-note", items: noteItems("tread", ["F","G","A"]), catchesForBoss: 8 },
      { name: "High Steps",  icon: "🎶", type: "name-note", items: noteItems("tread", ["G","A","B"]), catchesForBoss: 8 },
      { name: "Full Staff",  icon: "📖", type: "name-note", items: noteItems("tread", ["E","F","G","A","B","C","D"]), catchesForBoss: 20 },
    ]
  },
  {
    name: "Bass Bog", subtitle: "Read Bass Clef", masterTitle: "Bass Master",
    zones: [ ... same 4-zone shape with "bass" prefix, G-A-B / A-B-C / B-C-D / G-FE... ]
  },
  {
    name: "Grand Valley", subtitle: "Mixed Clefs", masterTitle: "Grand Master",
    unlock: { needsMaster: [0, 1] },
    zones: [ ... clef: "mixed", pool = union of treble + bass, prefixed "mix" ... ]
  },
];
```

Only `makeQuestion` / `staffSVG` / `buildAnswers` / `announceQuestion` are clef-aware; catch, gym, master, and Dex stay generic over `items` exactly like PianoWild.

> `noteItems` and the letter-pool helpers should be **generalized**: PianoWild's `clusterZones` hardcodes the C-D-E/E-F-G/G-A-B/full-C–B ladder. Reuse its *shape* (4 zones, 3 sub-pools + full, `catchesForBoss` 8/8/8/20) but parameterize the letter list per region so treble and bass get their own octaves without editing the helper.

## Staff drawing

PianoWild's `staffSVG(letter)` is **treble-hardcoded** (treble glyph + y-map keyed to treble positions). NoteWild needs a clef parameter:

```js
staffSVG(letter, { clef: "treble" | "bass", compact: false })
```

- **Clef glyph** — treble path already exists in PianoWild (`<path .../>` at pianowild.html:393). Add a **bass/F-clef** glyph (rounded double-dot "C" with two dots around the F line).
- **y-map** — PianoWild's `ys` map positions letters relative to the E line (treble). Define parallel maps:
  - treble: E=bottom line, F=bottom space, G=2nd line, A=2nd space, B=3rd line, C=top space (middle C ledger above), D=top line.
  - bass: G=lowest line… E=two spaces below... (bass notehead coordinates).
- **Ledger lines** — treble: middle-C note above the staff needs a ledger (already handled for `letter === 'C'` in PianoWild). Bass: any note outside the five lines (e.g., ledger up to G above bass staff) needs the same treatment.
- **Compact mode** — keep the `compact:true` variant for the answer-choice grid.

Every `staffSVG` call site must pass the current zone/region's clef. Store it on the item metadata: `ITEM_META[id].clef`.

## Question flow (`name-note` only)

One question type. In `makeQuestion(id)`:

```js
const meta = ITEM_META[id];           // { type, regionIndex, zoneIndex, clef }
const note = noteLetter(id);          // 'C'..'G'
const gymPrompt = current.mode === 'master' ? "Master trial! 5 in a row!"
                   : current.mode === 'boss' ? "Gym! 3 in a row!" : null;
return {
  id, type: "name-note", note,
  prompt: gymPrompt || "What note is this?",
  speakText: "What note is this?",
  speakLetter: false,
  answer: note,
  display: "staff",
  clef: meta.clef,                   // "treble" | "bass" | "mixed" (resolved at draw time)
};
```

- **`buildAnswers`** — letters = the zone's pool (max ~7). Same grid as PianoWild's letter-choices, but **no piano HTML** (drop the `usesPiano` branch entirely).
- **Mixed region** — for a mixed round, resolve `clef` to `treble` or `bass` at question time (`Math.random()` per question); the notehead is drawn in that clef and the letter comes from that clef's octave. Same answer letter set is fine because the pool is the union and letters are unambiguous once the clef is shown.
- **Audio** — on correct, play the real pitch. Extend `playPitch`'s `FREQ` map from treble-anchored C4 (PianoWild hardcodes C4=261.63) to a map keyed by **clef+letter** so bass-note G sounds G2 (98 Hz), not G4. Easiest: `FREQ[bassNote] = FREQ[trebleNote] / (power of 2)` from whichever reference octave.

## Region / gate change: "both clefs to unlock mixed"

PianoWild unlocks region N when region N-1's master is cleared (`highestUnlockedRegion`). NoteWild needs a **different gate**:

- Region 0 (treble) and region 1 (bass) are **both unlocked at start** (both available from the map immediately).
- Region 2 (mixed) unlocks **iff** `masterCleared[0] && masterCleared[1]`.

So replace `highestUnlockedRegion` logic with:

```js
function regionPlayable(ri){
  if(ri === 0 || ri === 1) return true;
  if(ri === 2) return !!state.masterCleared[0] && !!state.masterCleared[1];
  return false;
}
```

`renderMap` still shows the locked-mixed teaser tile until both masters are cleared. `viewingRegion` can freely navigate 0/1 while both are open; only 2 is gated.

## Save data

Key: `notewild-save` (change from `pianowild-save`). Same shape, plus nothing else — the gate is recomputed from `masterCleared`:

```json
{
  "items": { "tread:E": 3, "bass:G": 1, "mix:C": 0 },
  "regionUnlocked": 2,
  "zoneUnlockedInRegion": { "0": 3, "1": 2, "2": 0 },
  "bossCleared": { "0-0": true, "1-0": true },
  "masterCleared": { "0": true, "1": false }
}
```

No migration logic — renamed item keys start fresh at 0 (same policy as PianoWild README).

## File layout (target — does not exist yet)

```
note-wild/
  notewild.html      # single file: markup + CSS + JS, mirrors pianowild.html
  README.md          # how to play + build notes (mirrors piano-wild/README.md)
```

Plus edits to existing files (do **after** `notewild.html` works):

- `index.html` — add a fourth `.game.note` card linking to `note-wild/notewild.html` (new color vars `--note-*`, new inline SVG logo: a bass clef notehead on a small staff).
- `README.md` (repo root) — add NoteWild to the intro list and the `## Games` bullet.
- `apple-touch-icon.png` (repo root) — leave shared for now; NoteWild can reuse the hub icon unless we want a per-game icon (PianoWild currently reuses the hub `apple-touch-icon.png` pattern — confirm against `word-wild/` before deciding; `number-wild/` ships its own `apple-touch-icon.png`, so per-game icons are already supported).

## Build order

1. Copy `piano-wild/pianowild.html` → `note-wild/notewild.html`.
2. Strip out the keyboard: `pianoHTML`, `usesPiano` branches, `find-key`/`staff-to-key`/`key-to-staff` `makeQuestion`/`buildAnswers` cases, `unlockKeys`/`lockKeys`, piano CSS (`.white-key`, `.black-key`, `.piano`, `.has-piano`).
3. Generalize `staffSVG(letter)` → `staffSVG(letter, { clef, compact })`; add bass-clef glyph; add ledger handling for out-of-range notes in both clefs.
4. Rewrite `REGIONS` to the 3-region / clef-prefixed layout above; store `clef` in `ITEM_META`.
5. Rework the unlock gate to the "both masters → mixed" rule (replace `highestUnlockedRegion` / `clampRegionUnlocks` / `regionPlayable`).
6. Re-key the audio: `FREQ` and `playPitch` by clef+letter so bass notes sound an octave (or two) lower.
7. Change the storage key to `notewild-save`.
8. Update the map screen header to show the clef-mode legend ("Treble · Bass · Mixed 🔒").
9. Write `note-wild/README.md` mirroring `piano-wild/README.md`.
10. Add the hub link in `index.html` and the bullet in the root `README.md`.
11. Smoke test in a browser: play treble zone 1 to clear a gym, beat both masters, confirm the mixed tile unlocks.

## Open questions for the user

- **Mixed-region note pool**: union of a full treble octave and a full bass octave (12 letters, some repeated letter/different octave) — or start mixed with a smaller "common notes" pool (say C-D-E in both) and expand per zone?
- **Bass-clef glyph quality**: hand-drawn path (like PianoWild's treble) or use a Unicode ♭ glyph on a staff with a proper font? PianoWild drew its treble by hand; a bass clef by hand is less forgiving.
- **Shared apple-touch-icon**: per-game icon (like `number-wild/`) or share the hub icon (like `piano-wild/` currently does)?
- **Do we want a "listen" affordance** — a small speaker the kid can tap once more to re-hear the pitch — matching the piano-wild UX where each question speaks/plays then waits?
