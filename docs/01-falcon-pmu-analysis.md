# Falcon / PMU analysis: the "falcon ucode table" @ file 0xfee4

**Analysis target:** the 256 bytes reported as a "falcon ucode table" at file offset
`0xfee4` (image offset `0xf4e4`) of the P100 GP100 vBIOS ROM.

**ROMs:** `p100-gpu0.rom` (276,992 B, vBIOS 86.00.4D.00.01, build 2017-11-13,
subsystem `10DE:118F`); `gpu1/2/3` (rev `0x84` build, 272,896 B, byte-identical
perf tables). All offsets are file offsets unless noted (file = image + 0xA00).

---

## Verdict

The 256 bytes at `0xfee4` are **not a firmware image**. They are an arbitrary
256-byte window taken from the middle of a large **encrypted/signed opaque
partition** (≈ `0xfe54`–`0x26000`) that carries the board's protected firmware
manifest — the PMU/ACR falcon code region. There is no magic, no structure:
entropy is 6.96 bits/byte (window) and 7.4–7.9 bits/byte (surrounding partition).
**No NVIDIA falcon ucode magic exists anywhere in the ROM.**

## Evidence

1. **Magic scan of the full 276 KB ROM:** 0 hits for the ucode signature
   `7e e1 ac 00` (`0x00ace17e`), `de e0 ac 00`, ELF `\x7fELF`, `PMUX`, `NVPM`,
   or `FALCON`. The `0xfee4` window itself: entropy H = 6.96 bits/byte, only
   2 zero bytes.

2. **Surrounding structure** (image #2, starts at file `0xfe00` = image offset `0xf400`):

   | File offset | Bytes / meaning |
   |---|---|
   | `0x0fe00` | `55 aa 85 00 f1 0e 00 00 0b 00 64 86 01 00` — image data header (`0x85` = vBIOS rev) |
   | `0x0fe1c` | PCIR: vendor `0x10de`, device **`0x15f8` (P100)**, rev `0x0085`, class `0x030000` |
   | `0x0fe40` | **NPDE** descriptor: `4e 50 44 45 00 01 10 00 85 00 00 00 0b 00 03 00 a8 07 01 00 30 fb 01 00` — tag `NPDE`, ver 0.1, hdr `0x10`, rev `0x85`, opaque u32s `0x17a8` / `0x1fb30` |
   | `0x0fe54` | payload begins `24 60 8d ee fb 58 d2 7a 75 e3 b9 99 …` (144 B), then the 256 B "falcon data" at `0xfee4` (`5f 1d af dd e3 b3 4b a8 73 74 67 73 …`), then continues; contiguous max-entropy run to ≈`0x11600` and structurally similar entropy out to `0x26000` |

3. **The pointer hunt:** the only occurrence of the u16 `f4e4` (`0x0d89`) is an
   x86 `81 fa f4e4` *cmp* instruction displacement inside PREOS code — not a
   table pointer. The BIT table (`0xbb0`, v0.1, 18×7-byte entries) is the PREOS
   **code** vector table: e.g. `'D'` (`0x44`) → `0x1a00`, `'P'` (`0x50`) → `0x222c`
   — both x86 code. No falcon/ucode BIT table entry exists, and the BMP signature
   (`ff 7f NV 00`) is absent from this ROM.

4. **Cross-card diffs:** cards 1/2/3 (same build, rev `0x84`) have byte-identical
   `0xfe00`–`0x26000` partitions, including the 256 B at `0xfee4`; per-card unique
   bytes appear only in the InfoROM/serial area (`0x37088`+, not covered here).
   Card 0 (rev `0x85`) differs from card 1's partition almost entirely
   (99.4% of bytes, XOR entropy 8.0) → per-build distinct encrypted content,
   **no keystream reuse** (XOR-crib attack ruled out).

5. **NPDS/NPDE descriptor formats:** `NPDS` (`de10 c015` | rev | `0xe0`) + `NPDE`
   appear in 3 formats (v0.0 with a big opaque payload, v1.0, v1.1 with only
   zero/version records). The NPDE header rev always matches that image's PCIR
   revision (`0x7a`/`0x85`/`0x84`/`0x6f`/`0x42`/`0x63`) → **NPDE = NVIDIA PROM
   Data Entry**, a per-image manifest header whose v1.0 form fronts the encrypted
   manifest region.

## What the falcon (PMU) is on GP100, and the lock question

- The GP100 PMU is a **RISC-V Falcon** at MMIO `0x10a000`. In nouveau, GP100
  (`nv130`) wires `pmu = gm200_pmu_new`, whose only firmware interface is
  `gm200_pmu_nofw` ("firmware unavailable") → **the OS driver never loads PMU
  ucode**; the vBIOS **PREOS code boots the PMU firmware during POST**, before the
  OS. RM/nouveau then only *talk to* the PMU (RPC via msgq/cmdq).
- PMU duties: DVFS clock switching (core + HBM2/EMC), automatic/power brake,
  thermal & power-cap governor, pstate transitions. Fan control on Pascal is left
  to software by the open stack (`nvkm_pmu_fan_controlled`).
- **ACR** (GM2xx/GP1xx variant, `gm200_acr`): the HS/FWSEC + PMU LS images live in
  a Write-Protected Region (WPR). The `wpr_header` status codes
  (`VALIDATION_CODE_FAILED`, `VALIDATION_DATA_FAILED`, `REVOCATION_CHECK_FAILED`, …)
  plus FWSECLIC RSA-2048 (256 B) signatures show that the **ucode images
  themselves** are signed/validated. That is the lock the falcon side enforces —
  over **its own code** — and it is exactly what the opaque encrypted partition at
  `0xfe54`–`0x26000` belongs to.
- Performance/Boost/OC tables (e.g. the perf table at image `0xba5a`) are parsed
  by host-CPU code (RM on the CPU; P100 has no GSP) and fed to the PMU as runtime
  RPC arguments. **Nothing in the falcon path signs or hashes those tables**; a
  modified perf table is indistinguishable from a genuine one at the falcon.

### Conclusion on the "vBIOS hard stop"

The falcon/PMU does **not** enforce a lock over table data. It enforces (via ACR +
the encrypted NPDE manifest region) integrity of the PMU/ACR **firmware image**
only:

- Tampering inside the `0xfe54`–`0x26000` partition → failed validation /
  undecryptable firmware → **dead card**. This is the real "hard stop".
- A zeroed OC table in the perf tables → **in-band policy**, not falcon
  enforcement — and therefore safely editable. See
  `02-perf-table-analysis.md` for the table-level fix.

## Artifacts (not redistributed in this repo)

Two extraction artifacts were produced during analysis; their binaries are not
included here, only their identities for provenance:

| Artifact | Coverage | SHA-256 |
|---|---|---|
| `falcon_256_0xfee4.bin` (256 B) | the window at `0xfee4` | `8045367f8d8f0d9e9c990b5756448a0914bad4f2328467f6e7422aac204a6dd7` |
| `opaque_blob_image2.bin` (8704 B) | region `0xfe00`–`0x12000` | `53976acc1fa397677c59b76f1c5e3908310dacd91812f8ca82a676df99041a5c` |
