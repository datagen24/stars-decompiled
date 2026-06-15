# STARS.EXE Binary — Static Facts

> Confirmed by Ghidra headless analysis. The binary itself is **never committed** — it's
> commercial software; locate your own copy of `STARS.EXE` (Stars! 2.60j retail) before
> reproducing.

## Identity
| Field | Value | confidence |
|-------|-------|------------|
| File | `STARS.EXE` (MD5 `d0264285536ac7a837a8b049ad0df898`) | confirmed |
| Format | **New Executable (NE)** — 16-bit Windows | confirmed |
| Language/proc | `x86:LE:16:Protected Mode` | confirmed |
| Version | Stars! **2.60j** (header stamp `0x2A60`) | confirmed (version); decode hypothesis |
| Functions | 1354 | confirmed |
| Symbols | 10545 | confirmed |
| Imports | KERNEL, USER, GDI, COMMDLG, TOOLHELP, WIN87EM (resolved via Ghidra win16 export symbols) | confirmed |

## Segment layout
- **36 code segments** `Code1..Code36` (`1000:`–`1118:`), each ≤ ~38 KB (16-bit 64 KB cap; the read
  primitive's 30000-byte chunking matches this scale).
- **1 data segment** `Data37` @ `1120:` (~26 KB) — holds global state, string tables, the cipher
  seed table (`DS:0x49`) and runtime LCG state (`DS:0x224a`, `DS:0x224e`).
- **~120 resource segments** `Rsrc*` (bitmaps/sounds/dialogs) — not relevant to save format.
- `EXTERNAL` thunk block `14f8:` (imports). Decompiler warns "non-existing memory at 14f8:*" — these
  are import thunks, harmless.

## Function map (save/load + cipher)
Segment:offset addresses; names are Ghidra auto-names (`FUN_seg_off`).

| Address | Role | confidence |
|---------|------|------------|
| `1040:414c` | Raw block-read primitive (`_lread` in ≤30000-byte chunks until a 32-bit length is exhausted) | confirmed |
| `1070:370e` | Read-or-copy-from-cache wrapper (reads from in-memory buffer via `1118:0f3a` memcpy, else `_lread`) | confirmed |
| `1070:356e` | **Per-block reader**: read 2-byte plaintext header → `size=word&0x3ff`, `type=word>>10`; read payload; type 8 → key-init, else → decrypt | confirmed |
| `1040:1922` | **Key schedule** — seed LCGs from salt table + warm-up from gameId/player/turn | confirmed |
| `1040:1a7a` | **Decrypt loop** — keystream XOR over payload (4 bytes/PRNG call) | confirmed |
| `1040:19a8` | **PRNG core** — L'Ecuyer combined dual-LCG, returns `s1-s2` | confirmed |
| `1118:0c28` | signed 32-bit divide helper (Schrage `s/q`) | confirmed |
| `1118:0cc2` | 32×32 multiply helper (Schrage `a*…`) | confirmed |
| `1118:0f3a` | far `memcpy` (word copy + segment-wrap handling) | confirmed |
| `1070:054a` | Large load/save dispatcher (~55 KB decompiled) — walks block stream | hypothesis (role) |

See `encryption.md` for the cipher and `file-format.md` for the block format.

## Reproducing
- Use Ghidra 11+ (Apple Silicon native). The NE format loader is built in.
- Headless analysis with `analyzeHeadless` works; the cipher functions are easy to spot
  via the prime constants `2147483563` / `2147483143` and the multipliers `40014` / `40692`.
- Cross-reference call sites of `_lread` (segment `1040`) to find the I/O layer; `FUN_1070_356e`
  is the entry point worth understanding first.
