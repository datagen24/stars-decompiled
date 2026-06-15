# Planet Blocks (types 7, 13, 14)

> Status: **CONFIRMED layout** — verified against decrypted real saves (1133+ planet-info instances
> across two unrelated games; type-7 walked in both `.xy` files). Offsets are into the **decrypted
> payload** (after the 2-byte block header word); integers little-endian; bitfields LSB-first.
> Key cross-cutting finding: in types 13/14 the optional sub-blocks are **flag-gated** (by PlanetInfo
> bits and ownership), and several `Structure*.xml` entries that list them as unconditional
> are corrected below.

## Type 7 — Planets / Universe definition (`.xy`)
Universe parameters + game name, immediately followed by `planetCount * 4` **raw (unencrypted)**
planet-position records.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 4 | unknown / id-ish | hypothesis | varies per game |
| 4 | 2 | universeSize (enum index) | confirmed | small enum, not raw bits (4 = largest seen) |
| 6 | 2 | density (enum index) | confirmed | |
| 8 | 2 | playerCount | confirmed | 16 / 8 in the two corpus games |
| 10 | 2 | planetCount | confirmed | 800 / 288; drives the raw tail length |
| 12 | 4 | startingDistance (enum index) | community | value 1 |
| 16 | 2 | gameSettings flags | community | bitfield (max-minerals, slow-tech, accel-BBS, public-scores, clumping, …) |
| 18 | 2 | unknown | hypothesis | 0 |
| 20 | 12 | per-player / victory data | hypothesis | structured, not decoded |
| 32 | 32 | gameName (ASCII, NUL-terminated) | confirmed | bytes after NUL are uninit buffer |

> No `Structure7.xml` exists in this repo. `starsapi-python/blocks/PlanetsBlock.py` offsets match
> real `.xy` files; this repo's `StarsHostEditor` `XY.cs` uses *different* offsets — that is
> `StarsHostEditor`'s own intermediate format, **not** the real `.xy`; do not use it as the spec.

### Planet record (4 bytes, raw, in the type-7 tail)
Packed 32-bit LE word giving the planet's name index and **delta-encoded** map position.

| Bits | Field | confidence | notes |
|------|-------|------------|-------|
| 31..22 (`>>22`) | nameId | confirmed | 10-bit index into Stars!' built-in planet-name table (1..998, unique per game) |
| 21..10 (`>>10 & 0xFFF`) | y | confirmed | absolute y, 12-bit |
| 9..0 (`& 0x3FF`) | xOffset | confirmed | 10-bit **delta from previous planet's x**; running x starts at 1000 |

Planet `id` = record index (0-based); displayed id = id+1. The `.xy` stores only the `nameId`; the
name text comes from the engine's built-in **`PLANET_NAMES` table (999 entries, 0-indexed by
`nameId`)**, **not** a string block. Authoritative table: this repo's
`StarsHostEditor/AtlantisSoftware/PlanetNames.cs` — verified by lookup
(e.g. `nameId 627` → "Nirvana"). A native reimplementation should embed this table to render planet
names. confidence: confirmed (data).

**Validation:** type-7 payload exactly 64 bytes in both `.xy`; tail = `planetCount*4` lands on EOF;
800+288 records decode to x/y inside the universe bounding box with unique nameIds.

## Type 13 — Planetary Information (full)
Variable-length full planet state (host knows all; a player knows owned + fully-scanned planets).

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | PlanetID (bits 0-10) + OwnerID (bits 11-15) | confirmed | owner 0-15, **31 = nobody** |
| 2 | 2 | PlanetInfo flags | confirmed | bit7 Homeworld, bit9 Starbase, bit10 Terraformed, bit11 Installations, bit12 Artifact, bit13 SurfaceMinerals, bit14 RouteTo |
| 4 | 1 | DepletionLength | confirmed | 2 bits each iron/bor/ger → byte widths of next field |
| +var | | mineral depletion (Fe/B/Ge) | community | widths from DepletionLength |
| +3 | 3 | Fe/B/Ge concentration | confirmed | sane 1-117 |
| +3 | 3 | current Gravity/Temp/Radiation | confirmed | sane 1-99 |
| if bit10 | 3 | original Grav/Temp/Rad | community | present iff Terraformed |
| if owner≠31 | 2 | EstimatedPop (12b) + unk(4b) | confirmed (corrected) | **gated on ownership**, not unconditional. Value is in **400-colonist blocks** (×400 = est. colonists); used for the scanned/estimated figure |
| if bit13 | 1 | SurfaceLength (2b ×4: Fe/B/Ge/Pop) | confirmed (corrected) | **gated on SurfaceMinerals bit13** |
| +var | | surface Fe/B/Ge/Population | confirmed (data) | widths from SurfaceLength. **SurfacePopulation × 100 = exact colonists** (own planet); a homeworld at turn 0 shows 250 → 25,000 = standard start |
| if bit11 | 8 | installations | **confirmed (data)** | gated on bit11 — sub-layout below |
| if bit9 | 4 | starbase slot+damage, mass-driver target+speed | community | gated on bit9 |
| if bit14 | 2 | RouteTo planet no. | community | gated on bit14 |

> Corrections to `Structure13.xml`: `EstimatedPop` and `SurfaceLength` are **conditional** (ownership
> / bit13), not unconditional. Unowned planets with leftover surface minerals (bit13, owner 31) carry
> the surface block but no EstimatedPop.

### Installations sub-layout (8 bytes) — `confirmed (data)`
Matches `Structure13.xml`, **verified against a single home planet over all 56 turn
backups**: mines and factories are both strictly monotonic and stay under their operating caps
(`floor(pop/10000 × {mineMax,facMax})`), defenses rise and pin at the **100** defense cap.
**Cross-confirmed against the in-game planet summary** (turn 56): decoded mines 244 / factories 236 /
defenses 100 / pop 460,700 / minerals 3000·3032·677 all match the panel exactly (Mines "244 of 460",
Factories "236 of 691" — the "of" values are the operating caps).

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 1 | ExcessPopulation | confirmed (data) | colonists over capacity (fluctuates turn-to-turn) |
| 1 | 12b | Mines (count) | confirmed (data) | low 12 bits of bytes 1–3 (LSB-first) |
| +12b | 12b | Factories (count) | confirmed (data) | high 12 bits of bytes 1–3 |
| 4 | 12b | Defenses (count) | confirmed (data) | bytes 4–5; caps at 100 |
| +12b | 4b | unused | hypothesis | 0 in all samples |
| 6 | 1 | flags | community | bit0 NoScanner, bit7 ContributeOnlyLeftoverResources |
| 7 | 1 | unknown | hypothesis | 0 in all samples |

The decoded mine/factory counts are the inputs the resource-production rules need:
`factoriesOperating = min(factories, floor(pop/10000 × facMax))`.

**Validation:** 1108 instances across 4 files; **consumed == size** for all; owners 0-15/31, IDs and
hab/minerals all sane.

## Type 14 — Partial Planetary Information (scanned/foreign)
Reduced record for planets seen only by scanner; most fields gated by a type-14-specific flag.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | PlanetID (11) + OwnerID (5) | confirmed | 31 = nobody |
| 2 | 2 | PlanetInfo flags | confirmed | **bit1 = HabAndMineralConcentrationsIncluded** (the type-14 gate); bits 7/9/10/11/13/14 as in type 13 |
| if bit1 | 1+var | DepletionLength + concentrations + current Grav/Temp/Rad | confirmed | entire hab/mineral block gated on bit1 |
| if bit10 | 3 | original Grav/Temp/Rad | community | Terraformed |
| if owner≠31 **and** bit1 | 2 | EstimatedPop | confirmed (corrected) | conjunction; `iif(owner=31,0,2)` in XML is wrong |
| if bit13 | 1+var | SurfaceLength + surface minerals/pop | confirmed | |
| if bit11 | 8 | installations | community | |
| if bit9 | 1 | starbase slot id | confirmed | **1 byte only** here (vs 4 in type 13) |

> Corrections to `Structure14.xml`: EstimatedPop gate is `owned AND habIncl`; minimal foreign records
> are 4 bytes (header+flags); starbase contributes 1 byte (not 4); no RouteTo field.

**Validation:** 25 instances; **consumed == size** for all, from 4-byte minimal to 20-byte full-hab.

## Sources
- This repo's `Structures/Structure13.xml`, `Structure14.xml`;
  `starsapi-python/blocks/PlanetsBlock.py`; this repo's `…/Planet.cs`.
- Field-by-field validation against decrypted `.xy`, `.m1`, and `.hst` files from two unrelated games.
