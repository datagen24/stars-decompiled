# Race Block Specification (block type 6 — "Race Details")

> Status: **layout CONFIRMED** — the field map below is from `Structure6.xml` (this repo's
> `Structures/`) and **verified field-by-field against decrypted real saves** (an `.r1` race file,
> the same player's `.m1` from a different game, and an unrelated game's `.m1`): every field
> parses and the cursor lands exactly on the block end (`consumed == size`). The race name
> `StarsText` encoding is also **confirmed** (`strings.md`).
> Offsets below are **within the decrypted type-6 payload** (i.e. after the 2-byte block header word);
> all multi-byte integers little-endian. The block is encrypted in the file — decrypt first
> (`encryption.md`).

## FullRace conditional
Offset 6 carries a **FullRace** bit (bit 2). When set (1), the entire definition is present (as in
`.r` race files and a player's own `.m` file). When clear (0), the block **skips from Homeworld to
NameLength** — only the identity + name are present (a thin reference to another player). The map
below is the **FullRace = 1** layout. All three validated files have FullRace = 1.

## Field map (FullRace = 1)
| Off | W | Field | confidence | notes / observed |
|:---:|:-:|-------|------------|------------------|
| 0 | 1 | PlayerID | confirmed | 0-based; `0xFF` on an unassigned `.r` file |
| 1 | 1 | ShipSlotsUsed | confirmed | design slots in use |
| 2 | 2 | PlanetCount | confirmed | `& 0x3FF` |
| 4 | 2 | Fleet/StarbaseDesign counts | confirmed | FleetCount = bits 0..11, StarbaseDesignCount = bits 12..15 |
| 6 | 1 | flags | confirmed | bits 0-1 = 3 (constant); **bit 2 = FullRace**; bits 3-7 = Logo (race icon) |
| 7 | 1 | Unknown (=1 in `.m`, 0 in `.r`) | hypothesis | "appears to require 1" |
| 8 | 2 | Homeworld | confirmed | planet id, `& 0x3FF` |
| 10 | 2 | Rank | community | |
| 12 | 4 | PasswordHash | confirmed | 0 when no password (one observed game = `0x00136B59`) |
| 16 | 1 | Gravity centre | confirmed | habitability — see below |
| 17 | 1 | Temperature centre | confirmed | |
| 18 | 1 | Radiation centre | confirmed | |
| 19 | 1 | Gravity low | confirmed | |
| 20 | 1 | Temperature low | confirmed | |
| 21 | 1 | Radiation low | confirmed | |
| 22 | 1 | Gravity high | confirmed | |
| 23 | 1 | Temperature high | confirmed | |
| 24 | 1 | Radiation high | confirmed | |
| 25 | 1 | GrowthRate (%) | confirmed | direct percent (observed 6, 12) |
| 26 | 6 | Tech levels E/W/P/C/El/B | confirmed | 1 byte each (observed `[3,3,6,4,4,1]`) |
| 32 | 24 | ResearchPointsSincePrevLevel (6 × u32) | confirmed (data) | per-field accumulated research toward next level; 0 in `.r`. Resets at level-up when it crosses `levelCost` |
| 56 | 1 | ResearchPercentage | confirmed | budget % to research (15, 86 observed) |
| 57 | 1 | Research priorities | confirmed | current = bits 0-3, next = bits 4-7 |
| 58 | 4 | ResearchPointsPreviousYear (u32) | confirmed (data) | research resources spent last turn (grows with the empire) |
| 62 | 1 | PopulationEfficiency | confirmed | colonists per resource (×100) |
| 63 | 1 | FactoryEfficiency | confirmed | resources produced / 10 factories |
| 64 | 1 | ResourcesToBuildFactories | confirmed | |
| 65 | 1 | FactoriesOperated | confirmed | per 10k colonists |
| 66 | 1 | MiningEfficiency | confirmed | |
| 67 | 1 | ResourcesToBuildMines | confirmed | |
| 68 | 1 | MinesOperated | confirmed | |
| 69 | 1 | SpendLeftoverPointsOn | confirmed | enum (0-3 observed) |
| 70 | 6 | Research cost E/W/P/C/El/B | confirmed | 1 byte each, value 0-2: **0=expensive (×1.75), 1=standard, 2=cheap (×0.5)** — direction confirmed via a controlled custom race |
| 76 | 1 | **PRT** | confirmed | primary trait index — see enum (HE=0, AR=8 observed) |
| 77 | 1 | Unknown (unused) | hypothesis | 0 in all samples |
| 78 | 2 | **LRT bitfield** | confirmed | one bit per lesser trait — see map |
| 80 | 1 | Unknown | hypothesis | 0 in all samples |
| 81 | 1 | flags2 | community | bit 5 = ExpensiveTechStartsAt3, bit 7 = FactoriesCost1LessGerm |
| 82 | 2 | MT items owned | community | 12 Mystery-Trader item bits (+4 unused) |
| 84 | 28 | Unknown ×14 (u16) | hypothesis | runtime/uninvestigated |
| 112 | 1 | PlayerRelationCount | confirmed | (0 in `.r`; 16 / 8 in `.m`) |
| 113 | n | PlayerRelation[count] | community | 1 byte each (neutral/friend/enemy) |
| … | 1 | NameLength | confirmed | bytes of encoded singular name |
| … | n | Name (StarsText) | confirmed | length-prefixed; see `strings.md` |
| … | 1 | PluralNameLength | confirmed | |
| … | n | PluralName (StarsText) | confirmed | |

> Evidence: a `Structure6` walker against three independent decrypted files; all three consume to
> exactly `size`. Tech "points-since-prev-level" and "previous-year" fields are runtime state and
> read 0 in the `.r` race file, non-zero in `.m` turn files — consistent with their meaning.

## Primary Racial Traits (PRT) — index
Single byte at offset 76. **HE=0 and AR=8 confirmed from data**; the rest are community-ordered and
consistent (no conflict observed).

| Idx | Trait | | Idx | Trait |
|:---:|-------|---|:---:|-------|
| 0 | Hyper Expansion (HE) | | 5 | Space Demolition (SD) |
| 1 | Super Stealth (SS) | | 6 | Packet Physics (PP) |
| 2 | War Monger (WM) | | 7 | Interstellar Traveller (IT) |
| 3 | Claim Adjuster (CA) | | 8 | Alternate Reality (AR) |
| 4 | Inner Strength (IS) | | 9 | Jack of All Trades (JOAT) |

## Lesser Racial Traits (LRT) — bitfield (offset 78, u16)
Confirmed: matches `Structure6.xml`, and decodes to valid trait sets on real races
(`0x1331` → IFE/GR/UR/CE/OBRM/BET; `0x248D` → IFE/ARM/ISB/NRSE/NAS/RS).

| Bit | Trait | | Bit | Trait |
|:---:|-------|---|:---:|-------|
| 0 | Improved Fuel Efficiency (IFE) | | 7 | No Ramscoop Engines (NRSE) |
| 1 | Total Terraforming (TT) | | 8 | Cheap Engines (CE) |
| 2 | Advanced Remote Mining (ARM) | | 9 | Only Basic Remote Mining (OBRM) |
| 3 | Improved Starbases (ISB) | | 10 | No Advanced Scanners (NAS) |
| 4 | Generalized Research (GR) | | 11 | Low Starting Population (LSP) |
| 5 | Ultimate Recycling (UR) | | 12 | Bleeding Edge Technology (BET) |
| 6 | Mineral Alchemy (MA) | | 13 | Regenerating Shields (RS) |
| | | | 14-15 | unused |

## Habitability encoding (offsets 16-24)
Three axes (gravity, temperature, radiation), each stored as **3 bytes: centre, low, high** (the
centres are grouped first at 16-18, then lows at 19-21, then highs at 22-24).
- **Immune** to an axis is the sentinel **`0xFF`** in that axis's bytes (model as a distinct enum
  case, never a magic number). Observed: one race immune on all three (all `0xFF`); another (AR)
  immune on gravity+radiation, temperature centre 87 / low 77 / high 97.
- Non-immune axes give a tolerance band `[low, high]` around `centre` on Stars!' 0-100 "click" scale.
  confidence: confirmed (sentinel + band layout); exact click→displayed-value mapping = hypothesis.

## Research cost (offsets 70-75)
One byte per field (Energy, Weapons, Propulsion, Construction, Electronics, Biology). Three settings
with values **0 = "costs 75% extra" (×1.75), 1 = standard (×1.0), 2 = "50% less" (×0.5)** — direction
**confirmed** by a controlled custom race that set fields expensive/expensive/cheap/cheap/standard/standard
and decoded to bytes `[0,0,2,2,1,1]`. "Starts at tech 3" is a *separate* flag (`flags2` bit 5,
offset 81), **not** encoded in these bytes — correcting earlier hypotheses that packed cost as 2
bits/field with a "starts at 3" bit.

## Race name (`StarsText`) — CONFIRMED
Both names are length-prefixed (`NameLength` byte, then encoded bytes; singular then plural) and use
the **StarsText** encoding — see `strings.md`. The codec round-trips byte-identically across
files belonging to the same race (different salts, different keystreams).

## Open questions
1. Meanings of the 14 `Unknown` u16 fields (offset 84) and the offset-7/77/80 unknown bytes.
2. Habitability click-scale → displayed gravity/temp/rad value conversion.
3. `SpendLeftoverPointsOn` enum values.

## Sources
- This repo's `Structures/Structure6.xml` (field layout) and `…/AtlantisSoftware/Race.cs`
  (RaceID/Homeworld accessors).
- Field-by-field validation against three decrypted race blocks (an `.r1`, two unrelated `.m1`s).
- PRT/LRT naming cross-ref: StarsAutoHost wiki.
