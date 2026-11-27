# Performance / Overclocking / Memory-Clock tables: full parse + hardware-safe max-performance fix

**ROMs analyzed** (read-only, unmodified):

| Dump | Size | vBIOS | PCIR rev | Subsystem |
|---|---|---|---|---|
| `p100-gpu0.rom` | 276,992 B | 86.00.4D.00.01 | `0x85` | `10DE:118F` |
| `p100-gpu1.rom` | 272,896 B | — | `0x84` | `10DE:118F` |

Cards `gpu2`/`gpu3` have byte-identical perf tables to `gpu1`.
**All offsets below are file offsets.** BIT/PERF_PTRS pointers are image offsets;
file = image + `0xA00`.

**Factory reference:** NVIDIA P100 **PCIe 16 GB** = base **1190 MHz** / boost
**1329 MHz** / mem 715 MHz (1430 Mbps HBM2) / **250 W**. NVIDIA P100 **SXM2 16 GB**
(same GP100 die, 16 GB HBM2) = base 1328 / boost **1480 MHz** / **300 W**.

### Table locations

| Table | gpu0 img → file | gpu1 img → file |
|---|---|---|
| Performance Table | `0x0b06c` → **`0xba6c`** | `0x0b05a` → **`0xba5a`** |
| Virtual P-State Table | `0x0b172` → **`0xbb72`** | `0x0b160` → **`0xbb60`** |
| Memory Clock Table | `0x0b340` → **`0xbd40`** | `0x0b32c` → **`0xbd2c`** |
| Overclocking Table | `0x0cb3f` → **`0xd53f`** | `0x0ca8a` → **`0xd48a`** |
| Power Policy Table | `0x0c363` → **`0xcd63`** | `0x0c34f` → **`0xcd4f`** |
| Power Topology Table | `0x0c010` → `0xca10` | `0x0bffc` → `0xc9fc` |

---

## 1. Performance Table @ file 0xba6c (gpu0) / 0xba5a (gpu1)

Header (13 B, identical in both ROMs):

```
50 0a 03 04 06 0a 00 0f 00 01 ff 00 ff
ID='P'  ver=0x0a  …field layout not in any public spec (open-gpu-doc has no
GP100 perf-table spec; ogkm has no parser)…
```

Followed by a long zero-padded header block; the first populated record block
starts at **`0xbb33` (gpu0) / `0xbb21` (gpu1)**. Raw block (gpu0; gpu1 is the same
at −0x12):

```
0bb2c: 00 00 00 00 00 00 00 | 0f 08 ff 2a 03 2a 43 98 02 08 06 08 46 98 02
0bb3c: 08 06 08 46 98 02 cb 02 cb c2 b2 00 86 06 86 46 98 02 2a 03 2a 03
0bb4c: 44 01 58 02 58 02 96 00 1c 02 1c 02 87 00 2a 03 88 02 0e 01 03 00
0bb5c: 03 c0 00 00                          (→ VP table follows at 0xbb72)
```

Decoded field values (GP100 uses two clock encodings, both confirmed against the
card's factory spec):

| file gpu0 | file gpu1 | bytes | decode | meaning |
|---|---|---|---|---|
| `0xbb36` | `0xbb24` | `2a 03` | u16 = 810 | aux domain clock (P-state/vid clock) |
| **`0xbb39`** | **`0xbb27`** | `43 98 02` | u24 ÷128 = **1328.52** | **P0 boost max (mirror 1)** |
| `0xbb3f` | `0xbb2d` | `46 98 02` | u24 ÷128 = **1328.55** | **P0 boost max (mirror 2)** |
| `0xbb45` | `0xbb33` | `46 98 02` | u24 ÷128 = **1328.55** | **P0 boost max (mirror 3)** |
| `0xbb48/4b` | `0xbb36/39` | `cb 02` ×2 | u16 = 715 | **memory clock 715 MHz** |
| `0xbb4d` | `0xbb3b` | `c2 b2 00` | u24 ÷128 = 5700.9 | mem-domain aux value |
| `0xbb51` | `0xbb3f` | `46 98 02` | u24 ÷128 = **1328.55** | **P0 boost max (mirror 4)** |
| `0xbb54`, `0xbb56` | −0x12 | `2a 03` ×2 | 810, 810 | aux |
| `0xbb58` | | `44 01` | 324 | aux |
| `0xbb5a`, `0xbb5c` | | `58 02` ×2 | 600, 600 | aux |
| `0xbb5e` / `0xbb60` | | `96 00` / `1c 02` | 150 / 540 | aux |
| `0xbb64` | | `2a 03` | 810 | aux |
| `0xbb66` | | `88 02` | 648 | aux |
| `0xbb68` | | `0e 01` | 270 | aux |

(The 810/600/540/324/270/648/150 set is the per-domain P-state clock ladder —
idle/display domains; the 715 pair is the HBM2 clock; the four `98 02`-family u24s
are the P0 graphics boost ceiling, matching the official 1329 MHz boost exactly.)

## 2. Virtual P-State Table @ file 0xbb72 (gpu0) / 0xbb60 (gpu1)

Header gpu0 `20 12 01 0a 04 0a 00 02 01 04 06 03 ff ff …`, gpu1 `20 10 01 …`
(version 1.0; **the gpu0 header is 2 bytes longer than gpu1** — rev `0x85` vs
`0x84` — this is the only structural difference between the two cards' perf spans).

4 × 41-byte records, clock = u32 LE ÷ 32768 (Q15.16; matches the established
Pascal decoding convention):

| record | marker @ gpu0 / gpu1 | clock u32 @ gpu0 / gpu1 | raw | MHz |
|---|---|---|---|---|
| 0 | `0xbb84` / `0xbb70` | **`0xbb85` / `0xbb71`** | `4b c9 52 02` | **1189.57** (P0 base) |
| 1 | `0xbbad` / `0xbb99` | **`0xbbae` / `0xbb9a`** | `4b c9 52 02` | **1189.57** (base, 2nd pstate) |
| 2 | `0xbbd6` / `0xbbc2` | **`0xbbd7` / `0xbbc3`** | `61 4a 98 02` | **1328.58** (P0 boost max) |
| 3 | `0xbbff` / `0xbbeb` | **`0xbc00` / `0xbbec`** | `61 4a 98 02` | **1328.58** (boost, 2nd pstate) |

These Q15.16 records are the authoritative **max graphics clock** the driver will
allow at P0. 1189.57 ≈ official 1190 base; 1328.58 ≈ official 1329 boost. The
table is exactly at PCIe-P100 spec — the ceiling, not a bug.

## 3. Memory Clock Table @ file 0xbd40 (gpu0) / 0xbd2c (gpu1)

Header (identical modulo internal pointers): `11 1a 14 0c 08 04 …`
= version `0x11`, header size 26, base entry 20 B, strap record 12 B, 8 straps,
**4 entries** (GP100 HBM2 variant of the open-gpu-doc MemoryClockTable spec, which
lists strap size 26 for GDDR5). Only entry 3 is populated: **min = max = 1000**
(internal rate; operating bus is 715 MHz / 1430 Mbps per the perf block and
datasheet). Straps all `ff 00 00 50 …` (single HBM2 config).
**No memory headroom: 715 MHz is the HBM2 part rating. Do not touch.**

## 4. Overclocking Table @ file 0xd53f (gpu0) / 0xd48a (gpu1)

26 bytes, **byte-identical in both ROMs**:

```
10 06 05 02 05 02 | 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
ver 0x10 hdr 06   ^limits(5,2 / 5,2)   ^body: all-zero = NO user OC headroom
```

Body all zeros → `nvidia-smi -lgc/-rmc` offsets beyond stock and any third-party
OC are rejected. This is the **in-band half of the "vBIOS hard stop"** (the
datacenter lock). The table immediately abuts the DI/DT table at +0x1a, and both
P100s carry a complete **Fan Cooler / Fan Policy** table set (RPM targets up to
5000 RPM + temp→RPM curve) at `0xd419`+ — active fan control ⇒ **air-cooled
PCIe-style P100 boards (118F)**, 250 W-class.

## 5. gpu0 vs gpu1 diff (perf span, aligned −0x12)

Performance Table headers, the entire `0xbb33` boost block, and all four VP
records are **byte-identical**. Only differences: VPS header byte[1] (`0x12` vs
`0x10` = header length), the resulting 2-byte record shift, memclk internal
pointers, power-policy size bytes (`30 27 36` vs `30 26 2e`), and gpu0's BIT
living 0x30 earlier. **Clock/power caps are the same on all four cards.**

## 6. Power Policy Table @ file 0xcd63 (gpu0) / 0xcd4f (gpu1)

| Item | gpu0 | gpu1 | Value |
|---|---|---|---|
| Power slider **ENABLED** | `0xcd6c` | `0xcd58` | marker `0f 02 ff ff ff` |
| Default power limit (u32) | `0xcdfc` | `0xcdd7` | `90 d0 03 00` = **250,000 mW** |
| Max power limit (u32) | `0xce00` | `0xcddb` | **250,000 mW** |
| Power-brake constants | ≈`0xcdc8` | ≈`0xcda3` | `28 46 0f 00 28 46 0f 00` (left intact) |

Default = max = 250 W: `nvidia-smi -pl` has **zero** headroom. 250 W is the PCIe
board's rated TDP, so this number is correct for the hardware.

---

## 7. Where the performance is capped ("vBIOS hard stop"), precisely

1. **P0 boost ceiling = 1328.5–1328.6 MHz** — VP-table records @ gpu0
   `0xbbd7`/`0xbc00` (+ u24 mirrors @ `0xbb39`/`0xbb3f`/`0xbb45`/`0xbb51`) and the
   gpu1 counterparts. This is the NVIDIA **PCIe-bin** ceiling (1329), i.e. 151 MHz
   below the boost NVIDIA validated on the same GP100-16GB silicon in the SXM2
   P100 (1480 MHz).
2. **OC table body = 0** (`0xd53f`/`0xd48a`) — blocks any in-band clock offset.
3. **Power limit fixed at 250 W** (max == default) — correct for these 250 W
   boards; keep.
4. **Memclk 715 MHz** — part rating; keep.

## 8. Concrete hardware-safe fix (NOT applied — analysis only)

**Strategy:** raise the **P0 boost ceiling only**, to the highest clock NVIDIA has
validated for this exact die (GP100, 16 GB HBM2): **1480 MHz** (SXM2 P100 bin).
Every protective mechanism is left untouched:

- power limit stays 250 W → power brake still hard-caps board draw at design TDP,
- voltage policy/device/rail tables untouched → regulator window and AVFS curve
  unchanged,
- thermal slowdown/shutdown, power-brake and DI/DT tables untouched.

Because Boost 3.0 on GP100 *requests* clocks and AVFS/power-brake *grant* them,
raising the ceiling is **self-limiting**: a weak chip or a hot card simply never
reaches it; the card can never draw more than 250 W or run hotter than the
untouched slowdown table allows. Extra performance comes only from loads that were
previously clock-limited under the 250 W cap (the common case for LLM inference
on 4×P100).

### Byte patch (clock = 1480.00 MHz)

Encodings:
- u32 Q15.16: 1480 × 32768 = `0x02E40000` → LE `00 00 E4 02`
- u24 ÷128: 1480 × 128 = `0x02E400` → LE `00 E4 02`

**p100-gpu0.rom** (all sites verified present at analysis time):

| file offset | current bytes | new bytes | decode change |
|---|---|---|---|
| `0xbb39` | `43 98 02` | `00 e4 02` | 1328.52 → 1480.00 |
| `0xbb3f` | `46 98 02` | `00 e4 02` | 1328.55 → 1480.00 |
| `0xbb45` | `46 98 02` | `00 e4 02` | 1328.55 → 1480.00 |
| `0xbb51` | `46 98 02` | `00 e4 02` | 1328.55 → 1480.00 |
| `0xbbd7` | `61 4a 98 02` | `00 00 e4 02` | 1328.58 → 1480.00 (VP rec 2) |
| `0xbc00` | `61 4a 98 02` | `00 00 e4 02` | 1328.58 → 1480.00 (VP rec 3) |

**p100-gpu1.rom** (and gpu2/gpu3 — same table bytes; verify sites before patching):

| file offset | current bytes | new bytes |
|---|---|---|
| `0xbb27` | `43 98 02` | `00 e4 02` |
| `0xbb2d` | `46 98 02` | `00 e4 02` |
| `0xbb33` | `46 98 02` | `00 e4 02` |
| `0xbb3f` | `46 98 02` | `00 e4 02` |
| `0xbbc3` | `61 4a 98 02` | `00 00 e4 02` |
| `0xbbec` | `61 4a 98 02` | `00 00 e4 02` |

**Conservative first stage (recommended stepping):** 1450 MHz —
u32 `00 e8 d4 02` @ the VP sites, u24 `e0 d4 02` @ the perf-block sites. Test one
card, then step 1450 → 1480. The 1189.57 base records (gpu0 `0xbb85`/`0xbbae`,
gpu1 `0xbb71`/`0xbb9a`) are deliberately left at 1190 — they set the guaranteed
base, not the max.

### Explicitly DO NOT change

- `90 d0 03` power u32s (250 W = board rating; exceeding it stresses VRM/cooler
  beyond spec — the actual damage risk).
- Memory Clock Table (HBM2 has no headroom; overclock → corruption).
- Thermal policy / slowdown / shutdown, power-brake constants, DI/DT.
- Voltage Rail/Device/Policy tables (gpu0 image `0x4f1e`/`0x4eba`/`0x4f39` →
  file `0x591e`/`0x58ba`/`0x5939`).
- The falcon/PMU partition @ ≈`0xfe54`–`0x26000` (encrypted; tamper = dead card).
- OC-table body (no need — raising the ceiling makes in-band offsets unnecessary,
  and leaving it zero keeps `nvidia-smi` from letting a runaway offset exceed the
  tested window).

## 9. Other performance-relevant tables checked

- **Power Topology** (img `0xc010`/`0xbffc`) @ file `0xca10`: sensor weighting
  only — not a cap.
- **Thermal Policy** (img `0xca19`): present on both; values not fully decoded —
  intentionally untouched.
- **Voltage Rail/Device/Policy:** intact (regulator spec window).
- Perf-table header bytes `0f 08` and `0f 02` slider markers parsed as above.
- The BIT @ `0xbb0` on gpu0 is the PREOS code-vector BIT, not PERF_PTRS — the
  `perf_ptrs.json` pointers are authoritative (double-checked).

## 10. Flash & validation checklist (for the follow-up, if writes are ever authorized)

1. Patch copies only; keep pristine backups and confirm against `data/SHA256SUMS`.
2. `nvflash --protectoff`, flash one card, `nvflash -r 3` readback, diff vs the
   patched file (nvflash recomputes image checksums; the extracted dumps' PCI
   image-length/checksum fields are already bogus, so do **not** hand-fix them).
3. `nvidia-smi -q -d CLOCK` → Max Clocks should read 1480 MHz.
4. Validate under the card's **existing production workload** (e.g. the running
   llama-server inference job) while watching `nvidia-smi dmon -s pc`; do not run
   synthetic sustained burn-in benchmarks. Expect higher clocks on clock-limited
   kernels, power capped ≈250 W, slowdown untouched. Abort criteria: visual
   artifacts or Xid errors, or clocks oscillating with power-brake assert at
   temperatures above the slowdown threshold.

*Analysis performed on read-only copies; no ROM file was modified.*
