# Stars! File Format Specifications

This directory contains markdown specifications for the **Stars!** (1995, Empire Interactive
/ Mare Crisium) save-file format, recovered through:

1. **Static analysis** of `STARS.EXE` (Ghidra, Stars! 2.60j NE binary).
2. **Runtime/data analysis** of decrypted real saves (`.xy`, `.m*`, `.hst`, `.r*`).
3. **Cross-reference** with PaulCr's `Structures/*.xml` (in this repo) and the
   community decompiled utilities (this repo's `StarsHostCreator_StarsHostEditor`,
   `StarsHostEditor`, `StarsPlayerEditor` sources) and `stars-4x/starsapi-python`.

These specs **build on** PaulCr's XML structure files, not replace them. For each block
covered, the field map was verified field-by-field against decrypted real saves; where
the data disagreed with the XML the correction is documented inline (and the
corresponding XML in `../Structures/` has been amended with a clarifying comment).

## Confidence convention

Every fact carries one of:

- **`confirmed`** — agreed by at least two independent sources (e.g. our own dumps **and**
  the community XML/code, or two independent dumps).
- **`community`** — documented by the community (StarsAutoHost wiki, `Structures/*.xml`,
  decompiled utilities) but not (yet) re-derived from data.
- **`hypothesis`** — inferred/expected, not yet corroborated.

## Contents

| File | Covers |
|------|--------|
| [`file-format.md`](file-format.md) | File header (block type 8), block-stream framing, block-type enum, file-type extensions |
| [`encryption.md`](encryption.md) | The keystream cipher (L'Ecuyer combined dual-LCG, Schrage's method), key schedule, seed table |
| [`strings.md`](strings.md) | **StarsText** — the nibble-stream name encoding used for race/fleet/design names |
| [`binary.md`](binary.md) | `STARS.EXE` (Stars! 2.60j) static facts: identity, segment layout, function map for save/load + cipher |
| [`race.md`](race.md) | Block **type 6** — Race Details (full field map, PRT/LRT enums, habitability, research cost) |
| [`planets.md`](planets.md) | Block **types 7, 13, 14** — `.xy` universe definition, Planetary Information, Partial Planetary Information |
| [`fleets.md`](fleets.md) | Block **types 16, 17, 19, 20, 21** — Fleet Details, Partial Fleet, Waypoints (with/without task), Fleet Name |
| [`designs.md`](designs.md) | Block **types 26, 28, 29, 30, 31** — Ship/Starbase Design, Production Queues, Battle Plan, Battle Record header |
| [`blocks-misc.md`](blocks-misc.md) | Block **types 0, 9, 12, 40, 43, 45** — Terminator, X-File Info, Messages, Player Messages, Multi-object dispatch, Scores |

## Reading order

If you are new to the format, start with `file-format.md`, then `encryption.md`, then
`strings.md` — those three are prerequisites for parsing payload contents. After that the
per-block specs can be read in any order.

## Corrections to existing `Structures/*.xml`

Where our data disagreed with PaulCr's XML, the relevant XML has been updated with an
inline `<!-- NOTE … -->` comment summarizing the correction; the per-block markdown spec
carries the full explanation and evidence. Notable corrections:

- **Structure13.xml** — `EstimatedPop` and `SurfaceLength` sub-blocks are **flag-gated**
  (by ownership / `PlanetInfo` bit 13), not unconditional.
- **Structure14.xml** — `EstimatedPop` gate is `owned AND HabIncluded`; starbase contributes
  1 byte (not 4); no `RouteTo` field.
- **Structure19.xml** — record is **18 bytes** (8-byte header + 10-byte task params),
  not 8 bytes; `WarpSpeed` is in byte 7 low nibble (not byte 6).
- **Structure20.xml** — `WarpSpeed` is in byte 6 **high** nibble, not low nibble.
- **Structure21.xml** — not "Unknown"; it is the **fleet name** (length-prefixed StarsText).
- **Structure26.xml** — `TotalBuilt` and `TotalRemaining` are 32-bit LE; an
  earlier interpretation as 16-bit + unknown was incorrect.
- **Structure31.xml** — empty in the XML; header now recovered (`BattleNumber`,
  `RecordLength`, etc.), though the replay-token body remains opaque.

See the per-spec "Corrections" callouts for evidence and observed values.

## Validation corpus

The specs were validated against decrypted real saves from two unrelated games (one solo
campaign, one PBEM free-for-all), spanning `.xy`, `.r1`, `.m*`, `.h*`, and `.hst`. Each
spec lists the instance counts checked and notes whether `consumed == size` on every block.
The save files themselves are not included here — they are user-generated game data
that belongs to their authors.

## Acknowledgements

- **PaulCr** for the `Structures/*.xml` block-structure documentation — the starting point
  for everything here.
- The **stars-4x** community for the decompiled utility sources (this repo) and
  `starsapi-python`.
- The **StarsAutoHost wiki** for the file-extension / file-role overview, copy-protection
  notes, and PRT/LRT trait names.
