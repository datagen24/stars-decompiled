# File Format Specification

> Status: **MIXED** — the file-header block (type 8) is `confirmed` from byte analysis and
> from Ghidra (`STARS.EXE` 2.60j); the block-stream framing (6-bit type / 10-bit size in a
> single header word) is `confirmed`; per-block payloads are split between `confirmed`
> (race, planets, fleets, designs, scores, multi-object dispatch) and `community` / `hypothesis`
> for rarer blocks. Every fact below carries a `confidence:` tag.

## Confidence + promotion rule
- `confirmed` — agreed by **two independent sources** (e.g. community docs **and** a byte
  dump, or two independent dumps). Promote here only with evidence cited inline.
- `community` — documented by the community (StarsAutoHost wiki, `Structures/*.xml`,
  decompiled utilities) but not yet re-derived from data.
- `hypothesis` — inferred/expected, not yet corroborated.

---

## File types
The file's role is also stored *inside* the header as a `dt*` enum (see "File-header block",
offset 16). Extensions:

| Ext | Role | `dt` value | confidence |
|-----|------|:----------:|------------|
| `.xy` | Universe/master definition (planet positions, universe params); static for the game | `dtXY = 0` | confirmed (data+community) |
| `.x1`..`.x16` | Log/orders submitted player → host; carries copy-protection codes | `dtLog = 1` | community |
| `.hst` | Host file (authoritative game state); host-only, may be password-protected | `dtHost = 2` | community |
| `.m1`..`.m16` | Per-player turn file (race + empire state at turn start) | `dtTurn = 3` | confirmed (data+community) |
| `.h1`..`.h16` | Per-player history (accumulated view of the universe) | `dtHist = 4` | community |
| `.r1`..`.rN` | Race definition (Custom Race Wizard); `.r1` default, any N allowed | `dtRace = 5` | confirmed (data+community) |

> `N` = player number, **1..16** (community-confirmed; not 1..9 as sometimes hypothesized).
> `.pN`/`.pla` (planet dump) and `.fN`/`.fle` (fleet dump) are TSV text exports, not binary saves.

---

## Stream structure
A file is a flat sequence of typed **blocks**. The first block (type 8, the file header) is stored
**plaintext**; everything after it is **encrypted** (see `encryption.md`). confidence: confirmed (data)

```
[ block 0: File Header (type 8) ]   <- plaintext; carries gameId + salt
[ encrypted block ][ encrypted block ] ... until EOF
```

### Block header word
Each block begins with a single little-endian 16-bit word packing **type** and **payload size**:

| Bits | Field | Extract | confidence |
|------|-------|---------|------------|
| 15..10 (high 6) | `blockType` | `word >> 10` | confirmed (data+community) |
| 9..0 (low 10) | `payloadSize` (bytes following the header) | `word & 0x03FF` | confirmed (data+community) |

Evidence: block 0 word = `0x2010` → type `8`, size `16`. The 16-byte payload exactly spans offsets
2–17 (`J3J3` + gameId + version + 3 words), cross-checking the size decode. Max payload per block is
therefore 1023 bytes; larger structures span multiple blocks. confidence: confirmed (data)

**Important:** the header word is *never* encrypted — only the payload bytes are. This means
a parser can walk the block stream (i.e. determine `(type, size)` for every block) without
running the cipher. The cipher is only needed to materialize the payload *contents*.

---

## File-header block (block type 8) — CONFIRMED
18 bytes total (2-byte block word + 16-byte payload). Offsets are from file start; all multi-byte
integers little-endian.

| Offset | W | Field | Observed values | confidence |
|:------:|:-:|-------|-----------------|------------|
| 0 | 2 | block word `0x2010` → type 8, size 16 | constant, every file | confirmed (data+community) |
| 2 | 4 | magic ASCII `"J3J3"` (`4A 33 4A 33`) | constant, every file | confirmed (data+community) |
| 6 | 4 | **gameId** — random per-game id | e.g. `0x0004E567`, `0x04E92C4D`; unassigned `.r1` = `0x00000000` | confirmed (data) |
| 10 | 2 | **version stamp** (packed) | `0x2A60`, constant across two unrelated games & all file types | confirmed constant; bit-decode hypothesis |
| 12 | 2 | turn number | `.m1`=31, `.xy`=0, `.r1`=5, second-game `.m1`=29 | confirmed (data via cipher round-trip) |
| 14 | 2 | **salt word**: bits 5..15 = `lSaltTime` (11-bit), bits 0..4 = **playerIndex** (5-bit) | see below | playerIndex confirmed (data); salt community |
| 16 | 2 | **fileType + flags**: low byte = `dt*` enum, high byte = flags | see below | fileType confirmed (data); flags hypothesis |
| 18 | … | start of **encrypted** block stream | — | confirmed (data) |

**Version stamp.** `0x2A60` is constant across two unrelated games and all file types → it is the
Stars! engine version, not game data. The analyzed saves are Stars! **2.60j** (host log
format). Exact major/minor/increment bit packing is open. confidence: hypothesis (decode)

**Salt word (offset 14).** Looks random across files (`0x5880, 0x96BF, 0xABFF, 0xF740, 0x547F`) —
consistent with a time-derived salt seeding the cipher. The low 5 bits are the player index and
match file role exactly:
- `.m1` (`0x5880`) and another `.m1` (`0xF740`) → `& 0x1F = 0` → player 1 (zero-based 0).
- `.xy` (`0x96BF`, `0x547F`) and `.r1` (`0xABFF`) → `& 0x1F = 0x1F = 31` → "no player" sentinel.

**fileType (offset 16, low byte).** Matches the `dt*` enum across 5 files / 2 games: `.xy`→0,
`.m1`→3, `.r1`→5. One `.m1` has high byte `0x22` (flags set: e.g. "host using file" /
"multiple turns"); another has high byte `0x00`. confidence: confirmed (data) for the enum;
flag bit meanings hypothesis.

---

## Encryption / obfuscation
**See `encryption.md` for the full recovered cipher.** Summary:

- **Only block payloads are encrypted.** Every block's 2-byte header word (type/size) is
  **plaintext**, as is the entire file-header (type 8) payload.
- Evidence (data): walking a real `.m1` as `[plaintext 2-byte header][skip size]` chains
  cleanly from offset 0 to **exactly** EOF (47 blocks). Block structure is parseable
  **without** the cipher; the cipher is only needed to materialize payload *contents*.
- The cipher is a **keystream XOR** driven by **L'Ecuyer's combined dual-LCG** (Schrage's
  method), seeded from the salt via a 64-entry table and warmed up by a count from
  gameId/player/turn.

### Consequence: encrypted bodies cannot be byte-diffed pre-decryption
Comparing two consecutive-turn backups of the same `.m1` showed **563** differing byte
positions plus a length change. That is **not** real field churn: the salt (offset 14) is
regenerated every save, so the entire keystream changes and the ciphertext differs *everywhere*
even where the underlying plaintext is identical. Plaintext header diffing works fine
(that's how gameId, salt/player, and fileType were localized); encrypted-body diffing
requires recovering the cipher first.

---

## Copy protection (context for `.x` files)
`.x` (log/orders) files are stamped with two codes: the player's **serial code** and a
**hardware/machine code**. The host checks them at turn generation; matching serials with differing
hardware codes trip the protection (halved growth, lowered production, cancelled fleet orders). This
is per-`.x`-file and not retained turn-to-turn. Importers must tolerate/strip these when reading
legacy `.x` files. confidence: community

---

## Block-type enum (within-file blocks)
Distinct from the `dt*` *file* type. These are the 6-bit `blockType` ids inside the stream.
Source for most entries: `Structure*.xml` files in `../Structures/` (this repo).

| blockType | Name | Payload summary | confidence |
|:---------:|------|-----------------|------------|
| 0 | File Terminator / Checksum | empty — ends every file | confirmed (data) |
| 6 | Race Details | full race: hab, traits, production, names; see `race.md` | confirmed (data) |
| 7 | Planet / XY Data | universe params + planet count; 4-byte planet records appended outside payload | confirmed (data + code) |
| 8 | File Header | magic, gameId, version, salt+player, fileType+flags; **plaintext** | confirmed (data + community) |
| 9 | X-File Info | orders-file preamble (copy-protection codes) | community (Structure9.xml) |
| 12 | Stars Messages | in-game message log | community (Structure12.xml) |
| 13 | Planetary Information | full per-planet state (owner, pop, minerals, installations) | confirmed (data; XML had unconditional flags) |
| 14 | Partial Planetary Information | scanned/foreign planet state | confirmed (data; XML had unconditional flags) |
| 16 | Fleet Details | full fleet state | confirmed (data) |
| 17 | Partial Fleet Details | scan/intel view of a foreign fleet | confirmed (header); tail community |
| 19 | Waypoint With Task | waypoint orders (18 bytes; XML had only 8) | confirmed (data) |
| 20 | Waypoint (no task) | plain movement waypoint | confirmed (data) |
| 21 | Fleet Name | length-prefixed StarsText (XML labelled "Unknown") | confirmed (data) |
| 26 | Ship / Starbase Design Data | hull + component slots | confirmed (data) |
| 28 | Production Queue | host-side planet queue | confirmed (data) |
| 29 | Production Queue (x file) | player-submitted queue | community (no `.x` in corpus) |
| 30 | Battle Orders / Plan | per-race battle plans (7 per race) | confirmed (data) |
| 31 | Battle | resolved battle record; header recovered, replay body opaque | header confirmed (data); body hypothesis |
| 39 | Battle-Related (unknown) | unknown battle sub-record | community |
| 40 | Player Messages | inter-player message blocks | hypothesis (presence confirmed) |
| 43 | Multi-object | Minefields / Packets+Salvage / Wormholes / Mystery-Trader, dispatched by `idWord >> 13` | confirmed (data) |
| 45 | Scores | public score data (24 bytes) | confirmed (data) |
| 3, 4, 5, 21*, 23, 24, 32, 33, 35, 46 | Unknown / empty XML stubs | — | hypothesis |

> Numeric assignments come from `Structure*.xml` filenames and code references. The Ghidra
> dispatch table in `STARS.EXE` matches the observed type ids in the corpus.

### Per-block payload specs
Each payload below was mapped against the community `Structure*.xml` and **verified field-by-field
on decrypted real saves**. Detail lives in dedicated specs in this directory:

- **type 6** Race Details → `race.md`
- **types 7, 13, 14** planets / planet info → `planets.md`
- **types 16, 17, 19, 20, 21** fleets, waypoints, fleet names → `fleets.md`
- **types 26, 28, 29, 30, 31** designs, production queues, battle → `designs.md`
- **types 0, 9, 12, 40, 43, 45** terminator, x-info, messages, multi-object, scores → `blocks-misc.md`
- **string encoding** (race/fleet/design names) → `strings.md` (StarsText — confirmed)

---

## Open questions
1. **Version `0x2A60`**: exact major/minor/increment bit layout.
2. **Flags (offset 17 high byte)**: meaning of each bit (host-in-use, multi-turn, submitted, …).
3. **Block-type enum**: full numbering (the empty Structure XML stubs above).
4. Block ordering guarantees across types (fleet→waypoints→name is strict; cross-type ordering
   has not been characterized).

## Sources
- [Files Used in Stars! — Stars!wiki](https://wiki.starsautohost.org/wiki/Files_Used_in_Stars!)
- [Copy Protection Features — Stars!wiki](https://wiki.starsautohost.org/wiki/Copy_Protection_Features)
- [Patches and versions — Stars!wiki](https://wiki.starsautohost.org/wiki/Patches_and_versions)
- This repo's `Structures/*.xml` (PaulCr) and decompiled utility sources (`Decryptor.cs`,
  `Planet.cs`, `Fleet.cs`, `Design.cs`, `BattlePlan.cs`, `functions.cs`).
- Ghidra static analysis of `STARS.EXE` 2.60j (see `binary.md`).
- Byte-level validation of five decrypted real saves across two unrelated games.
