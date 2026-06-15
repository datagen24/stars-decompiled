# Design / Production / Battle Blocks (types 26, 28, 29, 30, 31)

> Status: **CONFIRMED** for 26/28/30 and the type-31 header (hundreds of real instances,
> `consumed == size`); type 29 is `community` (no `.x` files in the corpus); the type-31 replay body
> is an opaque token stream (`hypothesis`). Offsets into the **decrypted payload**; integers
> little-endian.

## Type 26 — Ship / Starbase Design
A flag word, hull id, picture, then either a full host record (build counts + component slots) or a
thin "unknown design" stub (what an enemy scanner sees).

**Flag word (offset 0-1):** bit 2 (`0x04`) = **FullDesign** (1 = full record, low byte `0x07`;
0 = unknown stub, low byte `0x03`); bits 8-12 = **DesignID** (`highByte & 0x1F`).

Common prefix:

| Off | W | Field | confidence |
|:---:|:-:|-------|------------|
| 0 | 2 | FlagWord (FullDesign bit2, DesignID) | confirmed |
| 2 | 1 | ShipHullID | confirmed |
| 3 | 1 | Picture/Icon | confirmed |

FullDesign = 1 tail:

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 4 | 2 | Armour | confirmed | |
| 6 | 1 | SlotCount | confirmed | number of component slots |
| 7 | 2 | Unknown16 | hypothesis | per-design serial/order index |
| 9 | 4 | TotalBuilt | confirmed | 32-bit LE (correction: XML splits as 2B+unknown) |
| 13 | 4 | TotalRemaining | confirmed | 32-bit LE |
| 17 | 4×SlotCount | ShipSlot[] | confirmed | see below |
| … | 1+N | NameLength + Name | confirmed | StarsText (`strings.md`) |

**Slot (4 bytes):** `ItemCategory` (u16, a **power-of-two category bitmask**, `0x0000` = empty),
`ItemID` (1, index within category), `ItemCount` (1).

Unknown (FullDesign = 0) tail: `Mass` (u16 @4), then `NameLength` + StarsText name. No slots/counts.

**Validation:** 326 full designs + 16 unknown stubs across the corpus (342 type-26 instances) all
`consumed == size`; hull ids, slot counts, build counts sane.

## Type 28 — Production Queue (host / `.hst` / `.m`)
Flat array of 4-byte packed items; item count = `size / 4` (0-length = empty queue).

| Bits (of 4-byte LE word) | Field | confidence | notes |
|------|-------|------------|-------|
| 0..9 | Count | confirmed | 0-1023 |
| 10..19 | Item | confirmed | 0-127 design index; 128+ = auto-build (128 mines … 136 terraform-class) |
| 20..31 | Completion | confirmed | partial-progress % of the front item |

**Validation:** every block a multiple of 4; counts ≤500, item ids sane; matches `Structure28.xml`.

## Type 29 — Production Queue (`.x` upload)
Same as type 28, prefixed with `PlanetID` (u16 @0), then `(size−2)/4` queue items.
**`community`** — no `.x` files in corpus; queue packing inherited from confirmed type 28.

## Type 30 — Battle Orders / Plan
One battle plan (a race has 7, ids 0-6). Fixed header + StarsText name.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 1 | RaceID (bits 0-3) + BattlePlanID (bits 4-7) | confirmed | 7 plans (0-6) per race |
| 1 | 1 | Tactic | confirmed | 0-6 (Disengage … Maximize damage) |
| 2 | 1 | PrimaryTarget (0-3) + SecondaryTarget (4-7) | confirmed | target class enum |
| 3 | 1 | AttackWho | confirmed | 0 Nobody / 1 Enemies / 2 Neutrals+Enemies / 3 Everyone / 4+ specific race |
| 4 | 1+N | NameLength + Name | confirmed | StarsText |

**Validation:** 131 plans (51 + 80 across the two corpus games) all `consumed == size`; matches
`Structure30.xml`.

## Type 31 — Battle Record / Replay
Self-describing header then a variable opaque token stream that drives the visual replay.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 1 | BattleNumber | confirmed | 1-based within the turn |
| 1 | 1 | Version/Tag | hypothesis | constant `0x0C` |
| 2 | 1 | unknown | hypothesis | constant `0x02` |
| 3 | 1 | ParticipantCount | confirmed | 2-6 |
| 4 | 2 | unknown | hypothesis | round count / flags |
| 6 | 2 | RecordLength | confirmed | == `size` for all 30 instances |
| 8 | 2 | unknown | hypothesis | covaries with length |
| 10 | 2 | LocationX | hypothesis | galaxy coord range |
| 12 | 2 | LocationY | hypothesis | |
| 14 | 2 | unknown | hypothesis | |
| 16+ | var | participant table + event tokens | hypothesis | opaque replay stream; not decoded |

> `Structure31.xml` is empty; header recovered from data (`RecordLength@6 == size` confirmed across
> 30 records).

**Validation:** 30 instances; `RecordLength == size` all; replay body left opaque.

## Sources
- This repo's `Structures/Structure26/28/29/30/31.xml`;
  `…/{Design.cs, DesignSlot.cs, Ships.cs, BattlePlan.cs}`.
- Validation against decrypted `.m1` and `.hst` files from two unrelated games.
