# Encryption / Obfuscation Specification

> Status: **CONFIRMED end-to-end.** Recovered from Ghidra static analysis of `STARS.EXE` 2.60j,
> cross-checked against this repo's decompiled `Decryptor.cs`
> (`StarsHostEditor/AtlantisSoftware/Decryptor.cs`) and `starsapi-python/encryption/decryptor.py`,
> and **validated by a full decrypt/re-encrypt round-trip on real saves** (see "Validation").
> Function addresses below are segment:offset in the NE image (see `binary.md`).

## Summary
Stars! enciphers **block payloads only**. Each block is `[2-byte plaintext header word][payload]`;
the header word (type/size) is **never** encrypted, but every payload except the file-header
block's is XORed with a keystream from a PRNG seeded off the file header. The PRNG is
**L'Ecuyer's combined dual-LCG** (Schrage's method).

This is the single biggest win for understanding the format: the cipher is *code that can be
read* (Ghidra + the community `Decryptor`), not memory that must be inferred.

## What is and isn't encrypted (CONFIRMED)
- **Plaintext:** every block's 2-byte header word; the entire file-header block (type 8) payload
  (magic, gameId, version, salt, fileType).
- **Encrypted:** the payload bytes of every non-header block.
- Evidence (code): `FUN_1070_356e` (per-block reader) reads the 2-byte header via the raw reader
  `FUN_1070_370e`, masks `size = word & 0x3ff` / `type = word >> 10` directly (no decrypt), then for
  non-header blocks calls the decrypt routine `FUN_1040_1a7a(payload, size)`.
- Evidence (data): walking a real `.m1` as `[plaintext header][skip size]` chains cleanly from
  offset 0 to **exactly** EOF (47 blocks, 1249 bytes). Header words are plaintext; only payloads
  are ciphertext. confidence: **confirmed (code + data)**

## Cipher: keystream XOR  (`FUN_1040_1a7a`)
```c
// decrypt(buffer, len): XOR a 32-bit-per-call keystream over the payload
for (p = buffer; p < buffer + (len & ~3); p += 2 words) {
    r = prng();            // 32-bit result (returned in AX:DX)
    p[0] ^= lo16(r);
    p[1] ^= hi16(r);       // 4 bytes consumed per prng() call
}
// trailing (len & 3) bytes are XORed byte-by-byte from one more prng() call
```
Encryption is the same operation (XOR is symmetric). confidence: confirmed (code)

## PRNG: L'Ecuyer combined dual-LCG  (`FUN_1040_19a8`)
Two 32-bit LCG states `s1`, `s2`, each advanced by **Schrage's method**, output is `s1 - s2`.
The math helpers are `FUN_1118_0c28` (signed 32-bit divide) and `FUN_1118_0cc2` (32×32 multiply).

```
step(s, a, q, r, m):           # Schrage: avoids 32-bit overflow
    s = a*(s % q) - r*(s / q)
    if (s < 0) s += m
    return s

prng():
    s1 = step(s1, a1, q1, r1, m1)
    s2 = step(s2, a2, q2, r2, m2)
    return s1 - s2             # 32-bit; split into two 16-bit XOR words
```

Recovered constants (from the immediates in `FUN_1040_19a8`/its callees):

| LCG | modulus `m` | multiplier `a` | `q = m/a` | `r = m%a` |
|----:|-------------|----------------|-----------|-----------|
| 1 | `0x7FFFFFAB` = 2147483563 | 40014 (`0x9C4E`) | 53668 (`0xD1A4`) | 12211 |
| 2 | `0x7FFFFF07` = 2147483143 | 40692 (`0x9EF4`) | 52774 (`0xCE26`) | **3791** |

> These are L'Ecuyer (1988): `a1/a2 = 40014/40692`, `q1 = 53668`, `q2 = 52774`. **The generator is
> deliberately "impure":** LCG2 uses the textbook `r2 = 3791` (not `m2 % a2 = 3535`) together with a
> Stars-specific modulus `m2 = 2147483143` (not the textbook 2147483399). Replicate exactly — these
> values are confirmed by the decompiled `Decryptor` and by the round-trip. confidence: confirmed

## Key schedule  (`FUN_1040_1922`)
Seeds the two LCG states from the **salt** and warms the generator by a count derived from
**gameId / player / turn**. Inputs come straight from the file-header fields (see `file-format.md`):
the caller `FUN_1070_356e` passes `gameId`, `salt>>5`, `turnWord`, `player = salt & 0x1f`, and a flag
bit (header offset 17, bit 4).

```
salt = lSaltTime                          # = header word@file-offset-14 >> 5 (11 bits)
i1 = salt & 0x1f                           # two 5-bit table indices...
i2 = (salt >> 5) & 0x1f
if (salt >> 10) & 1: i1 += 32             # ...+ bit-10 carry -> range 0..63
else:                i2 += 32
s1 = SEED_TABLE[i1]
s2 = SEED_TABLE[i2]
warmup = ((gameId & 3)+1) * ((turn & 3)+1) * ((player & 3)+1) + shareware
repeat warmup times: prng()               # advance before keystream use
```

**`SEED_TABLE` is a 64-entry table of small primes** (confirmed — the Ghidra `DS:0x49` read was a
misidentified location; the authoritative table is in the decompiled `Decryptor` and is validated by
the round-trip). Note index 55 is `279`, which is **not** prime — a quirk preserved verbatim:
```
  3,  5,  7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59,
 61, 67, 71, 73, 79, 83, 89, 97,101,103,107,109,113,127,131,137,
139,149,151,157,163,167,173,179,181,191,193,197,199,211,223,227,
229,233,239,241,251,257,263,279,271,277,281,283,293,307,311,313
```
Seeds are read as small ints; the warm-up rounds (1..65) diffuse them. `shareware` = bit 12 of the
word at file offset 16 (0 in all observed retail saves). confidence: **confirmed**

## Why every save differs entirely (ties to `file-format.md`)
The salt (`lSaltTime`) changes each save, so both LCG seeds and the warm-up count change → a
completely different keystream → every encrypted payload byte changes even when the plaintext is
identical. This is why raw byte-diffing of encrypted bodies is useless and the cipher recovery
had to precede any body-level diffing work.

## Validation (CONFIRMED)
A reference implementation of the algorithm above was run on five real saves across two unrelated
games (`.xy`, `.m1`, `.r1`, `.hst`). All three independent checks pass on every file:

1. **Round-trip:** decrypt → re-encrypt = **byte-identical** to the original (proves keystream
   consumption order and 4-byte padding are exact across every block).
2. **Decryption-dependent structure:** the `.xy` type-7 block decrypts to a sane `planetCount`
   (800 and 288 in the two games) and `planetCount*4` trailing planet bytes land the file on
   **exactly** EOF. A wrong keystream would yield garbage counts that miss EOF.
3. **Cross-file consistency:** two unrelated files belonging to the same race (an `.m1` and an
   `.r1`, different salts) decrypt their race-details (type 6) block to the **same** trailing
   signature (`…EMQgMo/EMQgMi`) — the same race recovered through two independent keystreams.

Header fields decoded along the way also matched the byte analysis (gameId, turn, player) — and
this is how **word A at file offset 12 was confirmed to be the turn number**.

## Implementation notes
- The cipher should be applied **per block payload**, not as a single post-header span — and
  the block-stream parser can read `(type,size)` from plaintext headers **without** running it.
- For tests / fixtures it can be useful to have an `IdentityCipher` that XORs with zeros so
  test data can be authored as plaintext.

## Sources
- Ghidra static analysis of `STARS.EXE` (MD5 `d0264285536ac7a837a8b049ad0df898`, Stars! 2.60j):
  `FUN_1070_356e`, `FUN_1040_1a7a`, `FUN_1040_19a8`, `FUN_1040_1922`, `FUN_1118_0c28/0cc2`.
- Round-trip validation against 5 decrypted real saves across two games.
- Authoritative cross-reference (in this repo):
  `StarsHostEditor/AtlantisSoftware/Decryptor.cs` and (external) `starsapi-python/encryption/decryptor.py`.
- L'Ecuyer (1988), "Efficient and Portable Combined Random Number Generators", *CACM* 31(6).
