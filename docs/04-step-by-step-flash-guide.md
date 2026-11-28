# Step-by-step: patching and flashing the P100 vBIOS

This guide walks through the full process — from backing up the original ROMs
to validating the patched card — in plain language. It is written for someone
who has never flashed a GPU before. Read it top to bottom before touching
anything.

> **Safety first:** vBIOS flashing is destructive if done incorrectly. A bad
> flash can brick the card. Every step below includes a verification check.
> If anything looks wrong, stop and back out — do not push through.

---

## What this does (and what it doesn't)

**What it does:** raises the P0 boost-clock ceiling from 1328.58 MHz (the
PCIe-bin ceiling) to 1480 MHz (the SXM2-bin ceiling for the same GP100 16 GB
HBM2 silicon). This is the only change.

**What it doesn't touch:**
- Power limit (stays 250 W — the board's rated TDP)
- Thermal slowdown/shutdown tables
- Voltage/rail/AVFS tables
- Power-brake constants
- DI/DT tables
- Memory clock (stays 715 MHz — the HBM2 part rating)
- The encrypted falcon/PMU partition (tampering here = dead card)

**Why it's safe:** Boost 3.0 on GP100 *requests* clocks and AVFS/power-brake
*grant* them. Raising the ceiling just means the card is *allowed* to go
higher — it doesn't *force* it. A weak or hot card simply never reaches
1480 MHz. The card can never exceed its 250 W design TDP or run hotter than
the untouched slowdown table allows.

---

## Prerequisites

Before you start, make sure you have:

1. **The original ROM files** — `p100-gpu0.rom`, `p100-gpu1.rom`,
   `p100-gpu2.rom`, `p100-gpu3.rom` (276,992 B and 272,896 B respectively).
   These are the read-only dumps used for analysis.
2. **The SHA-256 checksums** — see `data/SHA256SUMS`. You'll use these to
   verify the original files haven't changed.
3. **`nvflash`** — the NVIDIA Firmware Update Utility. It must be available
   on the host where the cards live.
4. **A backup plan** — you'll need to be able to restore the original ROM
   if something goes wrong.
5. **A quiet, stable environment** — no power interruptions during flashing.

---

## Step 1: Back up the original ROMs

Copy the original ROM files to a safe location. Do not work on the originals.

```bash
# Create a backup directory
mkdir -p ~/p100-vbios-backup

# Copy the original ROMs
cp p100-gpu0.rom p100-gpu1.rom p100-gpu2.rom p100-gpu3.rom ~/p100-vbios-backup/

# Verify the backups match the originals
cd ~/p100-vbios-backup
sha256sum -c /path/to/SHA256SUMS
```

If the checksums don't match, stop — something is wrong with the source files.

---

## Step 2: Verify the patch sites

Before patching, verify that the bytes at each patch site match what the
analysis found. This confirms the ROMs are the same files the analysis was
done on.

**p100-gpu0.rom** (file offsets):

| Offset | Expected bytes | New bytes |
|---|---|---|
| 0xbb39 | `43 98 02` | `00 e4 02` |
| 0xbb3f | `46 98 02` | `00 e4 02` |
| 0xbb45 | `46 98 02` | `00 e4 02` |
| 0xbb51 | `46 98 02` | `00 e4 02` |
| 0xbbd7 | `61 4a 98 02` | `00 00 e4 02` |
| 0xbc00 | `61 4a 98 02` | `00 00 e4 02` |

**p100-gpu1.rom** (and gpu2/gpu3 — same table bytes, verify before patching):

| Offset | Expected bytes | New bytes |
|---|---|---|
| 0xbb27 | `43 98 02` | `00 e4 02` |
| 0xbb2d | `46 98 02` | `00 e4 02` |
| 0xbb33 | `46 98 02` | `00 e4 02` |
| 0xbb3f | `46 98 02` | `00 e4 02` |
| 0xbbc3 | `61 4a 98 02` | `00 00 e4 02` |
| 0xbbec | `61 4a 98 02` | `00 00 e4 02` |

To verify, use a hex editor or a Python script. For example:

```python
with open('p100-gpu0.rom', 'rb') as f:
    rom = f.read()

sites = [
    (0xbb39, b'\x43\x98\x02'),
    (0xbb3f, b'\x46\x98\x02'),
    (0xbb45, b'\x46\x98\x02'),
    (0xbb51, b'\x46\x98\x02'),
    (0xbbd7, b'\x61\x4a\x98\x02'),
    (0xbc00, b'\x61\x4a\x98\x02'),
]

for offset, expected in sites:
    actual = rom[offset:offset+len(expected)]
    status = 'OK' if actual == expected else 'MISMATCH'
    print(f'0x{offset:05x}: {actual.hex()} (expected {expected.hex()}) — {status}')
```

If any site shows MISMATCH, stop — the ROM is not the same file the analysis
was done on. Do not patch.

---

## Step 3: Patch the ROMs

Create patched copies. Do not modify the originals.

```python
with open('p100-gpu0.rom', 'rb') as f:
    rom = bytearray(f.read())

# Patch the 6 sites
patches = [
    (0xbb39, b'\x00\xe4\x02'),
    (0xbb3f, b'\x00\xe4\x02'),
    (0xbb45, b'\x00\xe4\x02'),
    (0xbb51, b'\x00\xe4\x02'),
    (0xbbd7, b'\x00\x00\xe4\x02'),
    (0xbc00, b'\x00\x00\xe4\x02'),
]

for offset, new_bytes in patches:
    rom[offset:offset+len(new_bytes)] = new_bytes

with open('p100-gpu0-patched.rom', 'wb') as f:
    f.write(rom)
```

Repeat for gpu1/gpu2/gpu3 with the appropriate offsets.

**Conservative first stage:** if you want to be extra cautious, patch to
1450 MHz instead of 1480 MHz. The encoding is:
- u32 Q15.16: `1450 × 32768 = 0x02D40000` → LE `00 00 d4 02`
- u24 ÷128: `1450 × 128 = 0x02D400` → LE `00 d4 02`

Test one card at 1450 MHz first, then step to 1480 MHz if it works.

---

## Step 4: Verify the patched ROMs

After patching, verify the patched files:

```bash
# Check the patched bytes
python3 -c "
with open('p100-gpu0-patched.rom', 'rb') as f:
    rom = f.read()
sites = [0xbb39, 0xbb3f, 0xbb45, 0xbb51, 0xbbd7, 0xbc00]
for offset in sites:
    print(f'0x{offset:05x}: {rom[offset:offset+4].hex()}')
"

# Verify the patched files are the same size as the originals
ls -la p100-gpu0.rom p100-gpu0-patched.rom
```

The patched files should be the same size as the originals. The bytes at the
patch sites should match the new values.

---

## Step 5: Flash the card

This is the destructive step. Do it one card at a time.

```bash
# Disable vBIOS protection (required for flashing)
nvflash --protectoff

# Flash the patched ROM
nvflash -f p100-gpu0-patched.rom

# Read back the flashed ROM
nvflash -r 3 > p100-gpu0-readback.rom

# Diff the readback against the patched file
diff p100-gpu0-patched.rom p100-gpu0-readback.rom
```

If the diff shows differences, stop — the flash may have failed. Do not
proceed to the next card.

**Important:** `nvflash` recomputes image checksums. The extracted dumps'
PCI image-length/checksum fields are already bogus, so do not hand-fix them.

---

## Step 6: Validate

After flashing, validate the card:

```bash
# Check the max clocks
nvidia-smi -q -d CLOCK
```

The Max Clocks should read 1480 MHz (or 1450 MHz if you used the conservative
first stage).

Then run the existing workload (e.g., llama-server) and watch the clocks:

```bash
nvidia-smi dmon -s pc
```

Expect:
- Higher clocks on clock-limited kernels
- Power capped ≈250 W
- Thermal slowdown untouched

**Abort criteria:** artifacts/Xid errors, or clocks oscillating with
power-brake assert at temps above the slowdown threshold.

---

## Step 7: Rollback (if needed)

If something goes wrong, restore the original ROM:

```bash
# Disable vBIOS protection
nvflash --protectoff

# Flash the original ROM
nvflash -f ~/p100-vbios-backup/p100-gpu0.rom

# Read back and verify
nvflash -r 3 > p100-gpu0-rollback.rom
diff ~/p100-vbios-backup/p100-gpu0.rom p100-gpu0-rollback.rom
```

---

## Checklist

- [ ] Original ROMs backed up and checksums verified
- [ ] Patch sites verified byte-exact
- [ ] Patched ROMs created (copies, not originals)
- [ ] Patched ROMs verified (same size, correct bytes)
- [ ] One card flashed (not all four at once)
- [ ] Readback diff clean
- [ ] `nvidia-smi -q -d CLOCK` shows the new max clock
- [ ] Workload runs without artifacts/Xid
- [ ] Power capped ≈250 W
- [ ] Thermal slowdown untouched
- [ ] If anything goes wrong: rollback to original ROM

---

## What NOT to do

- Do not raise the power limit beyond 250 W — that stresses the VRM/cooler
  beyond spec and is the actual damage risk.
- Do not touch the memory clock — HBM2 has no headroom; over-clocking
  causes corruption.
- Do not touch the thermal policy/slowdown/shutdown tables.
- Do not touch the voltage rail/device/policy tables.
- Do not touch the falcon/PMU partition — tampering there = dead card.
- Do not flash all four cards at once — flash one, validate, then proceed.
- Do not skip the readback diff — it's the only way to confirm the flash worked.
