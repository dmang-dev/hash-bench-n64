# hash-bench-n64

Native **Nintendo 64** hashing-algorithm benchmark — **32 algorithms**
on a 93.75 MHz VR4300 (MIPS3, 64-bit) with `get_ticks_us()`
microsecond timing, displayed in libdragon's 64×28 software console.
Same algorithm set as
[`hash-bench-nds`](https://github.com/dmang-dev/hash-bench-nds) /
[`hash-bench-dsi`](https://github.com/dmang-dev/hash-bench-dsi) /
[`hash-bench-3ds`](https://github.com/dmang-dev/hash-bench-3ds) — the
"full" roster, since the VR4300 has native `uint64_t` and there's no
ROM-size constraint to exclude the heavy crypto algos.

| Tier | Algorithms |
|---|---|
| **Checksums (9)**       | CRC-8, CRC-16, CRC-32, **CRC-64**, Adler-32, Fletcher-{16,32,**64**}, Pearson-8 |
| **Non-crypto (11)**     | DJB2, FNV-1a, Knuth, Jenkins-OAT, PJW/ELF, SDBM, Murmur3-32, **Murmur3-128**, xxHash32, **xxHash64**, **SipHash-2-4** |
| **Cryptographic (12)**  | MD4, MD5, SHA-1, RIPEMD-160, SHA-256, **SHA-512**, **SHA-3-256**, **SHA-3-512**, BLAKE2s, HMAC-SHA256, **PBKDF2-HMAC-SHA256**, AES-CBC-MAC |

(Bold = exercises native 64-bit MIPS instructions where smaller
siblings have to synthesize them.)

[![ROM](https://img.shields.io/badge/ROM-prebuilt%20%26%20committed-success)](hash-bench-n64.z64)
[![Built with libdragon](https://img.shields.io/badge/built%20with-libdragon-orange)](https://github.com/DragonMinded/libdragon)
[![Format](https://img.shields.io/badge/format-Z64%20%281%20MiB%2C%20padded%29-lightgrey)](https://en64.shoutwiki.com/wiki/N64_ROM)

---

## What it does

On boot, libdragon's `console_init()` brings up a 640×240 16-bpp
display with a software-rendered 64×28 character grid. The harness
then:

1. Shows a startup banner with controls + emulator-compat notes.
2. Waits for **A** (so the user has time to verify the display is
   actually working before kicking off ~10 s of benchmarking).
3. For each algorithm, calls `get_ticks_us()` and loops the hash
   until 200 000 µs (200 ms) of wall time elapsed, recording
   `iters / elapsed_us / first 4 bytes of digest`.
4. Renders the full table in `RENDER_MANUAL` mode with one explicit
   `console_render()` per pass (avoids the per-printf vsync thrash
   that comes with `RENDER_AUTOMATIC`).

Sample output (from default `1024 B` mode, sorted by category):

```
hash-bench-n64 v0.2   VR4300 93.75MHz   1024 B baseline
sort=category   budget=200 ms
================================================================
 ALGO   TIER       ITER  US/IT  KB/s    H0 H1 H2 H3
----------------------------------------------------------------
.CRC8   checksum   1812    110  9075    DD ...
.CRC16  checksum    948    211  4744    F0 09 ...
.CRC32  checksum   3105     64 15564    7C 32 1B 5D
.CRC64  checksum   2440     82 12224    ...
.ADL32  checksum   2812     71 13735    13 D3 FE 10
.FLT16  checksum   2103     95 10522    D4 00 ...
.FLT32  checksum   2614     76 13093    F5 F3 FF 00
.FLT64  checksum   2256     88 11547    9C 5C 1D DD
.PRSN8  checksum   2256     88 11547    A0 ...
*KNUTH  non-crypto 1645    121  8417    16 9F A0 00
*OAT    non-crypto 2004     99 10256    D8 87 78 E4
... (full 32-row table; see screenshot in repo)
#MD4    crypto     1245    160  6371    A2 90 9A 64
#MD5    crypto      367    545  1878    63 B2 17 7A
#SHA1   crypto      233    861  1188    B6 67 86 CD
#RMD160 crypto      155   1298   788    21 D1 D6 3F
#SHA256 crypto      153   1315   778    8D 7E 56 67
#SHA512 crypto      270    742  1379    07 62 91 37
#SHA3-2 crypto       81   2477   413    D9 25 39 4C
#SHA3-5 crypto       44   4563   224    16 39 C3 62
#BLK2S  crypto      331    604  1692    AD 41 D5 E9
#HMACS2 crypto      122   1650   620    CE 13 93 93
#PBKDF2 crypto        6  37230    27    4F AC C7 38
#AESCBC crypto       43   4706   217    0E F7 EA 61
A:sort  B:rerun  Z:size-sweep  ST:rerun
```

Tier markers in the leading column: `.` = checksum, `*` = non-crypto,
`#` = crypto.

### Controls
- **A** — cycle sort (`category` → `by-speed` → `by-name`)
- **B** — re-run all algorithms
- **Z** — toggle single-size (1024 B) vs sweep mode (64 / 256 / 1024 B)
- **START** — re-run (libdragon has no clean exit; this is an alias
  for B)

### Buffer-size sweep mode

In sweep mode, every algorithm runs three times — once each with 64,
256, and 1024 byte buffers — and the table shows `KB/s @64`,
`KB/s @256`, `KB/s @1024` side-by-side. Useful for seeing
per-block setup amortization:

- **Crypto algos**: 64 B ≪ 256 B ≈ 1024 B (the first compress block
  dominates a 64-B input).
- **Per-byte algos**: 64 B ≈ 256 B ≈ 1024 B (linear, no setup cost).

---

## Try it

Pre-built ROM in repo root: [`hash-bench-n64.z64`](hash-bench-n64.z64)
(1 MiB, padded — see "Why padded" below).

**Recommended emulator: [Ares](https://ares-emu.net/)**. libdragon
ROMs use a *custom IPL3* (boot block) that bypasses the original N64
cartridge security check. Project64 versions before 4.x emulate the
CIC chip strictly enough to reject anything whose IPL3 doesn't match
a known commercial signature — they show a black screen forever.

Working alternatives:
- **[Ares](https://ares-emu.net/)** — most accurate modern N64 emu
- **[mupen64plus](https://mupen64plus.org/)** / **[m64p](https://m64p.github.io/)**
  (the GUI fork)
- **[cen64](https://cen64.com/)** — cycle-accurate, slow

The same `.z64` boots fine on real hardware via flash cart
(EverDrive 64, SC64) — the IPL3 issue is purely a Project64
emulation gap.

### Why the ROM is padded to 1 MiB

`n64tool` will produce a ROM at the actual byte count (~160 KB for
this project). Many emulators (notably Project64 < 4.x and some
flash-cart loaders) refuse sub-MiB ROMs since no commercial cart was
ever that small. We pad via `n64tool --size 1M` (added to
`N64_TOOLFLAGS` in the [Makefile](Makefile)). Padding is harmless on
accurate emulators and matches what real flash carts expect.

---

## Build from source

Requires the **mips64-elf gcc toolchain** plus a built+installed
**libdragon**. The Makefile expects `N64_INST` to point at the
toolchain prefix (with libdragon's headers under
`$(N64_INST)/mips64-elf/include/` and `libdragon.a` under
`$(N64_INST)/mips64-elf/lib/`).

The bundled [`build.bat`](build.bat) hard-codes `N64_INST=I:/libdragon`
for this machine — adjust to your install. On Linux/macOS / CI:

```
make N64_INST=/opt/libdragon
```

If you're starting from just the toolchain (no libdragon installed):

```bash
git clone https://github.com/DragonMinded/libdragon.git
cd libdragon
N64_INST=/opt/libdragon make libdragon
N64_INST=/opt/libdragon make install-mk install
# then build the host tools (n64tool, n64sym, n64elfcompress, mksprite, …)
# from tools/ — needs a host gcc (mingw64 on Windows, system gcc elsewhere)
N64_INST=/opt/libdragon make tools tools-install
```

`build.bat` also pre-renames a locked output `.z64` if an emulator
has it open — Windows won't let `make` delete a file held by
Project64/Ares, so we `move /Y` it aside before invoking `make`.

---

## Algorithms

All 27 algorithm `.c` files are byte-identical to
[`hash-bench-nds/source/`](https://github.com/dmang-dev/hash-bench-nds/tree/main/source).
The N64 main.c [`source/main.c`](source/main.c) is libdragon-native
(no console / joypad code shared with the ARM ports), but the
algorithm sources themselves are 100% portable C with `<stdint.h>`
only.

### Reference digests

Workload buffer: 1024 bytes of `(i * 31 + 7) & 0xFF` for
`i ∈ [0, 1024)`. HMAC-SHA256 key: ASCII `hash-bench-nds` + two zero
bytes (16 B total). PBKDF2: same key, 1000 iterations, 32-byte
output. SipHash-2-4 key: bytes `0x00..0x0F`. xxHash{32,64} seed: `0`.

The displayed `H0..H3` columns show the first four bytes of each
digest. Full reference values are documented in
[hash-bench-nds#reference-digests](https://github.com/dmang-dev/hash-bench-nds#reference-digests)
— they should match byte-for-byte across N64 / NDS / GBA / 3DS.

---

## Cross-platform comparison

The interesting thing about benchmarking the same algorithm set on
seven different CPU architectures is *which* algorithm wins changes
based on hardware capability. Specifically on the N64:

### SHA-512 beats SHA-256 (the opposite of every 32-bit platform)

| Platform | SHA-256 (KB/s) | SHA-512 (KB/s) | Ratio |
|---|---|---|---|
| GBA (ARM7TDMI 16.78 MHz) | ~280 | ~140 | SHA-256 wins 2× |
| NDS (ARM946E 33.5 MHz)   | ~620 | ~310 | SHA-256 wins 2× |
| **N64 (VR4300 93.75 MHz)** | **778** | **1379** | **SHA-512 wins 1.8×** |

SHA-256 uses `uint32_t` state; SHA-512 uses `uint64_t`. On every
32-bit platform, SHA-512's per-round cost doubles because each
operation synthesizes through two-register pairs. On the VR4300,
`daddu` / `dsrlv` / `dxor` are all single-cycle native instructions,
*and* SHA-512 processes 128-byte blocks vs SHA-256's 64-byte — so it
amortizes the round constant + message schedule cost over twice the
input. Net: SHA-512 wins by roughly the SHA-256-on-32-bit penalty
factor inverted.

### BLAKE2s leaves headroom on the table

BLAKE2s comes in at 1692 KB/s — faster than SHA-1 (1188 KB/s), as
expected. But the implementation uses a vanilla `ROL32` macro:

```c
#define ROL32(x, n) (((uint32_t)(x) << (n)) | ((uint32_t)(x) >> (32 - (n))))
```

GCC's `mips64-elf-gcc` lowers this to two shifts + an OR rather than
the single `rotr` instruction available on **MIPS32r2** — the VR4300
is **MIPS3** and predates `rotr`. Every BLAKE2s G() function does 4
rotations (×8 G calls per round × 10 rounds = **320 rotations per
64-byte block**), each costing ~3 cycles instead of 1. Hand-unrolling
G() and writing the rotations as inline asm would likely push BLAKE2s
past 2000 KB/s. Same applies to xxHash32, Murmur3-{32,128} — anything
rotation-heavy.

### AES-CBC-MAC at 217 KB/s

Pure-software AES with no T-table optimization (the implementation
prioritizes ROM size over speed — relevant on GB/GBA where ROM is
scarce). On the N64 we could afford the 4 KB of T-tables for ~3-5×
speedup. Future work.

---

## Timing methodology

libdragon's [`get_ticks_us()`](https://github.com/DragonMinded/libdragon/blob/trunk/include/n64sys.h)
returns a 64-bit microsecond counter derived from COP0 `count` (which
ticks at half the CPU clock = 46.875 MHz). The 64-bit width means it
*never* wraps in practice — at full rate it lasts ~600 000 years.

Per-algorithm budget: `BUDGET_US = 200000` (200 ms). The harness
keeps iterating until `get_ticks_us() - t0 >= BUDGET_US`, then
records the actual elapsed microseconds. PBKDF2-HMAC-SHA256 has a
single iteration that takes 37 ms (since one PBKDF2 invocation runs
1000 HMAC rounds), so it overshoots the budget naturally — that
overshoot is captured honestly in the `US/IT` column.

KB/s calculation, integer-only to avoid soft-FP cost:

```c
kbps = (iters * size_bytes * 1000) / elapsed_us;
//      ^^^^^^^^^^^^^^^^^^^^^^^^^^   bytes×ms — 64-bit intermediate
```

---

## Layout

```
source/                  Pure-C sources
  main.c                   libdragon console + joypad + bench harness
  crc8.c crc16.c crc32.c crc64.c                checksums
  adler32.c fletcher.c pearson.c                checksums
  djb2.c fnv1a.c tiny_hashes.c                  non-crypto
  murmur3.c murmur3_128.c xxhash32.c xxhash64.c non-crypto
  siphash.c                                     non-crypto (SipHash-2-4)
  md4.c md5.c sha1.c sha256.c sha512.c          crypto
  ripemd160.c sha3.c blake2s.c                  crypto
  hmac_sha256.c pbkdf2_sha256.c aes_cbc_mac.c   crypto
include/
  hashes.h                 32 function declarations (full roster)
build/                   gcc object files + maps + intermediate ELFs
Makefile                 libdragon n64.mk wrapper
build.bat                Windows wrapper (sets N64_INST + handles file locks)
```

All 27 `.c` algorithm files are shared verbatim with hash-bench-nds /
-dsi / -3ds. Only `main.c` is per-platform.

---

## Acknowledgments

- [libdragon](https://github.com/DragonMinded/libdragon) — N64
  homebrew SDK with custom IPL3
- [Ares](https://ares-emu.net/) — accurate N64 emulator that boots
  libdragon ROMs without complaint
- [DragonMinded](https://github.com/DragonMinded) and the libdragon
  contributors — for making N64 homebrew approachable in C
- [hash-bench-nds](https://github.com/dmang-dev/hash-bench-nds) — sibling
  project supplying the 27 algorithm sources
