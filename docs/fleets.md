# Fleet & Waypoint Blocks (types 16, 17, 19, 20, 21)

> Status: **CONFIRMED layout** for 16/19/20/21 (600+ real instances, `consumed == size`); type 17
> (scan view) is variable/intel-dependent and partly `hypothesis`. Offsets into the **decrypted
> payload**; integers little-endian. Block ordering is strict:
> **`16` (fleet) → one or more `19`/`20` (its waypoints, route order) → optional `21` (its name) →
> next `16`**.

## Type 16 — Fleet Details
Full owned-fleet record.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | FleetID (bits 0-8) + OwnerID (bits 9-12) + unk(3) | confirmed | `id = w & 0x1FF`, `owner = (w>>9)&0xF` |
| 2 | 4 | unknown (4 opaque bytes) | hypothesis | byte at offset 5 = flags; **bit3 = ShipCountWidthIs1** (1→counts are 1 byte, else 2) |
| 6 | 2 | PositionObjectID | confirmed | planet/object the fleet is at; 0 = deep space |
| 8 | 2 | X | confirmed | `0xFFFF` sentinel = orbiting/deep space |
| 10 | 2 | Y | confirmed | |
| 12 | 2 | ShipTypeBitmap | confirmed | bit N set ⇒ design slot N present (16 slots) |
| 14 | (1\|2)×popcount | per-slot ShipCount | confirmed | width per flags bit3; one entry per set bitmap bit, slot order |
| … | 2 | CargoLengthBitmap | confirmed | five 2-bit fields: Ironium, Boranium, Germanium, Population, Fuel |
| … | per-field | Fe/B/Ge/Pop/Fuel | confirmed | each width = `2^(len-1)` bytes (len 0 ⇒ absent) |
| … | 2 | DamagedShipTypeBitmap | confirmed (HST) | bit N ⇒ a 2-byte damage value follows for slot N |
| … | 2×popcount | per-damaged-slot damage | confirmed (HST) | |
| … | 1 | BattlePlan | confirmed (HST) | battle-plan index |
| … | 1 | WaypointAndTasksCount (trailing) | confirmed (consumed) | likely count of following 19/20 blocks (hypothesis on exact meaning) |

> In `.hst` host records the damage bitmap + battle-plan + trailing byte are always present
> (this repo's `Fleet.cs` hard-codes them) — not optional.

**Validation:** 362 instances (315 hst + 47 m1); `consumed == size` 100%; X/Y in galaxy range,
cargo/counts sane.

## Type 17 — Partial Fleet Details (scan/intel view)
Same header→ships→cargo backbone as type 16, but the trailing section is **scan-dependent**.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | FleetID(9)+OwnerID(4)+unk(3) | confirmed | same packing as type 16 |
| 2 | 4 | unknown | hypothesis | offset-5 byte here is only `0x00`/`0xFF` (not type-16 bit3 semantics) |
| 6 | 2 | PositionObjectID | confirmed | |
| 8 | 2 | X | confirmed | `0xFFFF` seen |
| 10 | 2 | Y | confirmed | |
| 12 | 2 | ShipTypeBitmap | confirmed | |
| 14 | ×popcount | per-slot ShipCount | hypothesis | width ambiguous in scan records |
| … | 2+var | CargoLengthBitmap + cargo | community | same encoding as type 16 for the parsed prefix |
| … | 0-5 | trailing intel bytes | hypothesis | varies with scanner penetration; no fixed tail |

**Validation:** 29 m1 instances; header/ships/cargo backbone walks, but trailing residue varies
(0-5 bytes) — confirms it is a reduced scan record, not the full type-16 tail. The only block not
fully closed in the corpus.

## Type 19 — Waypoint With Task (18 bytes, fixed)
A waypoint carrying an order/task and its parameters.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | X | confirmed | |
| 2 | 2 | Y | confirmed | |
| 4 | 2 | ObjectID | confirmed | target planet/fleet (0 for empty-space coords) |
| 6 | 1 | Task (low nibble) + flags (high nibble) | confirmed | task ∈ {1 Transport, 2 Colonize, 3 Remote-Mining, 5 Scrap, 6 Lay-Minefield, 8 Rout, …} |
| 7 | 1 | Warp (low nibble) + flags (high nibble) | confirmed | warp speed in low nibble |
| 8 | 10 | task parameters | confirmed (consumed) | order-specific (transport load/unload, etc.); per-field meaning hypothesis |

> Correction: `Structure19.xml` shows only 8 bytes — the record is **18 bytes** (10-byte param
> block follows). Warp is in byte 7 low nibble (not byte 6).

**Validation:** 207 instances, all exactly 18 bytes; X/Y sane; task codes map to the known order list.

## Type 20 — Waypoint (no task, 8 bytes fixed)
Plain movement waypoint.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 2 | X | confirmed | |
| 2 | 2 | Y | confirmed | |
| 4 | 2 | ObjectID | confirmed | |
| 6 | 1 | WarpSpeed (**high** nibble) | confirmed | `warp = byte >> 4`, 0-11; low nibble 0 |
| 7 | 1 | flags | confirmed (consumed) | repeat-count / waypoint-type (meaning hypothesis) |

> Correction: `Structure20.xml` says warp is byte-6 low nibble — it is the **high** nibble.

**Validation:** 492 instances, all exactly 8 bytes; warp 0-11.

## Type 21 — Fleet Name
Per-fleet custom name override, emitted after a fleet's waypoint chain.

| Off | W | Field | confidence | notes |
|:---:|:-:|-------|------------|-------|
| 0 | 1 | PayloadLength | confirmed | = `size − 1` |
| 1 | N | StarsText name | confirmed (structure) | decode via `strings.md` |

> Correction: `Structure21.xml` is an empty "Unknown" — it is the fleet name (length-prefixed
> StarsText).

**Validation:** 27 instances; `byte0 == size − 1` 100%; always positioned between a fleet's
waypoints and the next fleet.

## Sources
- This repo's `Structures/Structure16/17/19/20/21.xml`;
  `…/{Fleet.cs, Waypoint.cs, WayPointTask.cs}`.
- Validation against decrypted `.m1` and `.hst` files from two unrelated games.
