# Misc & Rare Blocks (types 0, 9, 12, 40, 43, 45 + rare)

> Status: 0 / 43 / 45 **CONFIRMED** from real data; 12 / 40 presence-confirmed with `hypothesis`
> layout; 9 `community` (only in `.x` files, none in corpus). Offsets into the **decrypted payload**;
> integers little-endian. Block header = one word, `type = word>>10`, `size = word & 0x3FF`.

## Type 0 — File Terminator / Checksum
Always the **last** block in every file; 2-byte payload that differs per file (e.g. `FF 31` in one
observed `.m1`), consistent with a trailing checksum/seal rather than data. confidence: terminator
role **confirmed**; value semantics `hypothesis`.

## Type 9 — X-File Info (`community`)
Orders-file preamble with copy-protection (machine + serial hashes). Not present in the corpus
(`.x` files only). Per `Structure9.xml`: length word, a packed 2×4-bit lengths byte, then ~15 hash
bytes (obfuscated).

## Type 12 — Stars Messages
In-game message log; one block per file, variable size. `Structure12.xml` is an empty stub.
Empirically a packed concatenation of short (~8-byte stride) numeric message records — message
bodies are generated from templates, **not** stored as text. confidence: presence confirmed,
layout `hypothesis`.

## Type 40 — Player Messages
Inter-player (PBEM) messages (found in two PBEM `.m*` files in the corpus). Each record: a small
header (length, then stable `1F 0A`, then from/to player words) and an obfuscated body. confidence:
header field *positions* stable across samples (`hypothesis`); semantics not confirmed.

## Type 43 — Multi-object (Minefields / Packets+Salvage / Wormholes / Mystery-Trader)
**Sub-kind = top 3 bits of the first word** (`TypeID = idWord >> 13`). Full records are **18 bytes**;
a **2-byte** payload is an ID-only deletion/empty stub. confidence: dispatch + record framing
**confirmed**.

| `idWord>>13` | sub-object | XML |
|:---:|-----------|-----|
| 0-1 | Minefields | Minefields.xml |
| 2-3 | Mineral Packets / Salvage | Packets.xml |
| 4 | Wormholes | Wormholes.xml |
| 6 | Mystery Trader | MysteryTrader.xml |

**43-Minefields** (18 B): ID word (MinefieldID:9, PlayerID:4, TypeID:3); XPos u16 @2; YPos u16 @4;
MineCount u32 @6; Unknown1 @10; flags @12 (type:2, exploding:1, …); Unknown3 @14; TurnNo @16.
*confirmed* (decoded a real record: X=1626, Y=2392, mines=279, turn=28).

**43-Packets/Salvage** (18 B): ID word (SalvageID:9, OwnerID:4, TypeID:3); X @2; Y @4;
Target+Speed @6 (PlanetID:10 — `0x3FF` = salvage — WarpSpeed-4:4, OverMD:2); Ironium @8; Boranium @10;
Germanium @12; Unknown @14-15; TurnNoLastChange @16. *confirmed* (minerals decode sanely).

**43-Wormholes** (18 B): ID word (WormholeID:12, TypeID:4); X @2; Y @4; Stability @6;
BeenThrough bits @8; CanSeeIt bits @10; TargetID (paired wormhole) @12; Unknown @14; TurnNo @16.
*confirmed header*; per-player bit semantics `community`.

**43-MysteryTrader** (18 B): per `MysteryTrader.xml` (ID, X, Y, XDest, YDest, Speed+unk, MetMT bits,
Items gift bitflags, TurnNo). No MT instance in corpus → `community`.

> Model suggestion: an enum with associated values
> `.minefield/.packetOrSalvage/.wormhole/.mysteryTrader` plus a `.deletionStub(id:)` case for the
> 2-byte form.

## Type 45 — Scores (24 bytes, fixed)
Public per-player/per-rank score record.

| Off | W | Field | confidence |
|:---:|:-:|-------|------------|
| 0 | 2 | PlayerID (bits 0-3) + VictoryCondition flag bits | confirmed (struct) |
| 2 | 2 | Rank | confirmed |
| 4 | 4 | Score | confirmed |
| 8 | 4 | Resources | confirmed |
| 12 | 2 | Planets | confirmed |
| 14 | 2 | Starbases | confirmed |
| 16 | 2 | UnarmedShips | confirmed |
| 18 | 2 | EscortShips | confirmed |
| 20 | 2 | CapitalShips | confirmed |
| 22 | 2 | TechLevels | confirmed |

VC bits (word 0, per `Structure45.xml`): owns-X-planets, attains-tech, exceeds-score,
exceeds-2nd-by-X, production-X, owns-X-capital-ships, highest-score-after-Y-years (individual bit
meanings `community`). **Validation:** 504 instances, every one exactly 24 bytes; ranks/scores
monotone.

## Rare / unseen types (Structure XML exists, no real instance in corpus)
| Type | Purpose | confidence |
|:----:|---------|------------|
| 3, 4, 5, 23, 24, 32, 33, 35, 46 | empty XML stubs | unknown |
| 39 | battle-related sub-record | community (layout unknown) |

## Sources
- This repo's `Structures/Structure0/9/12/40/43/45.xml` + `Minefields/Packets/Wormholes/MysteryTrader.xml`.
- Validation against decrypted `.m1`, `.m2`, `.m6`, `.h1`, and `.hst` files from two unrelated games.
