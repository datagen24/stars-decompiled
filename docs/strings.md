# StarsText String Encoding

> Status: **CONFIRMED** — recovered from the decompiled `functions.DecodeText` /
> `functions.EncodeText` in this repo
> (`StarsHostCreator_StarsHostEditor/AtlantisSoftware/functions.cs`) and validated
> byte-identically against decrypted real names from two unrelated games.

Race names, fleet names (block type 21), and design names (block type 26) are stored as **StarsText**:
a length-prefixed, nibble-stream code with shift escapes. Layout in a block is always
`[length: 1 byte][length encoded bytes]`.

## Algorithm
Expand the data bytes into a stream of 4-bit nibbles, **high nibble first** within each byte
(`0xC2` → `C, 2`). Walk the nibble stream:

| Nibble | Meaning | Nibbles consumed |
|:------:|---------|:----------------:|
| `0x0`..`0xA` | direct character `PRIMARY[n]` | 1 |
| `0xB`..`0xE` | shift: the **next** nibble indexes `TABLE[shift]` | 2 |
| `0xF` | literal ASCII byte, stored **low-nibble-first** (`byte = next2<<4 \| next1`) | 3 |

```
PRIMARY = " aehilnorst"          # index 0x0..0xA  (space + 10 most-common name letters)
TABLE 0xB = "ABCDEFGHIJKLMNOP"
TABLE 0xC = "QRSTUVWXYZ012345"
TABLE 0xD = "6789bcdfgjkmpquv"
TABLE 0xE = "wxyz+-,!.?:;'*%$"
```
A trailing partial escape (a `0xF`/shift nibble without its operands at end of stream) is padding —
ignore it. The decoded character count is **not** `2 × byteLength` because shifts/escapes consume
extra nibbles.

## Worked example
A race singular name stored as `C2 45 4D 51 67 4D 6F` → nibbles
`C 2 | 4 | 5 | 4 | D 5 | 1 | 6 | 7 | 4 | D 6 | F`:
`C2`→`S`, `4`→`i`, `5`→`l`, `4`→`i`, `D5`→`c`, `1`→`a`, `6`→`n`, `7`→`o`, `4`→`i`, `D6`→`d`,
trailing `F` = padding ⇒ **"Silicanoid"**. The plural ends `…4D 69`: the final `9`→`s` ⇒
**"Silicanoids"**.

## Validation
Decoding the type-6 race blocks of real saves yields sensible names with the structural tail
consuming exactly to block end. Examples from the corpus:

| Singular | Plural |
|----------|--------|
| Silicanoid | Silicanoids |
| Incognito | Incognitos |
| Dino | Dinosaurs |
| Meek | (plural verified separately) |

"Silicanoid" is a canonical Stars! AI race name — independent corroboration. Singular/plural are
independently stored strings (e.g. Dino → Dino**saurs**), not a mechanical "+s", and the codec
round-trips byte-identically.

## Notes
- The `0xD`/`0xE` tables and the `0xF` literal escape are taken verbatim from `DecodeText`/`EncodeText`
  (which agree) and confirmed by synthetic round-trip; no real name in the sample saves exercised
  digits 6-9 / `wxyz` / punctuation, so those rows are `code-sourced` rather than `byte-confirmed`.
- Encoding is the inverse; pad an odd nibble count with a trailing `F`.

## Sources
- `StarsHostCreator_StarsHostEditor/AtlantisSoftware/functions.cs`
  (`DecodeText`/`EncodeText`).
- Validation against decrypted race-detail blocks from two unrelated games.
