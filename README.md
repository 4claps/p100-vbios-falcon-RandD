# NVIDIA Tesla P100 (GP100) vBIOS — Reverse-Engineering Analysis

Static reverse-engineering analysis of four NVIDIA Tesla P100 PCIe vBIOS ROM dumps,
performed to understand the "vBIOS hard stop" that caps these cards below the
performance their silicon is capable of, and to specify a **hardware-safe change**
that lifts the boost-clock ceiling without touching any protection mechanism.

**Prepared by:** Juan, AI systems administrator.
**Task:** analyze NVIDIA Tesla P100 VBIOS ROMs to get past the vBIOS hard stop and
enable maximum performance.

**Tooling:**
- **REA** — a Ghidra-backed MCP (Model Context Protocol) server used for binary
  disassembly, structure recovery, and cross-referencing of the ROM images.
- Read-only EEPROM dumps taken with the NVIDIA Firmware Update Utility (`nvflash`).
- Cross-reference against the **nouveau** open-source kernel driver and NVIDIA's
  **open-gpu-kernel-modules** source (see `references/nouveau-reference.md`).

> **Scope & safety:** this is analysis *only*. All work was done on read-only
> copies of the ROMs; no ROM file was modified and nothing was flashed. The patch
> spec in `docs/02-perf-table-analysis.md` is a proposal, not an applied change.
> vBIOS flashing is destructive if done incorrectly — back up and verify first.

---

## Subject hardware

Four Tesla P100 PCIe 16 GB cards (PCI subsystem **10DE:118F**, GP100 die, 16 GB
HBM2), one air-cooled PCIe-class card per slot:

| Item | Value |
|---|---|
| vBIOS version (card 0) | `86.00.4D.00.01` (PCIR rev `0x85`), build date 2017-11-13, 276,992 B |
| vBIOS build (cards 1–3) | PCIR rev `0x84`, 272,896 B (perf tables byte-identical to card 0) |
| Subsystem ID | `10DE:118F` |
| UEFI guest ROM | version `0x3000B` (x64), GP1xx variant |
| InfoROM version | `H400.0201.00.08` (backup present) |

SHA-256 checksums of the analyzed ROM files: see `data/SHA256SUMS` (the ROM binaries
themselves are not redistributed here).

**Offset convention used everywhere in this repo:** BIT/PERF_PTRS pointers are
*image* offsets; in these extracted dumps, **file offset = image offset + 0xA00**.

---

## Key findings (summary)

1. **The "falcon ucode table" at file `0xfee4` is not a firmware image.** It is a
   256-byte window into a large **encrypted/signed opaque partition** (≈`0xfe54`–`0x26000`)
   that carries the board's protected PMU/ACR firmware manifest (NPDE descriptors).
   Zero falcon ucode magic exists anywhere in the ROM; entropy is 6.96–7.9 bits/byte;
   cross-card diffs show per-build encrypted content with no keystream reuse.
   Tampering inside that partition = undecryptable firmware = **dead card**.
   *(Details: `docs/01-falcon-pmu-analysis.md`.)*

2. **The falcon/PMU does not enforce any lock over performance-table data.**
   On GP100 the PMU is a RISC-V Falcon whose firmware is booted by vBIOS PREOS code
   during POST; the OS driver only talks to it via RPC. ACR (the `gm200_acr` variant)
   signs/validates **only the PMU/ACR firmware images** (WPR + FWSECLIC RSA-2048).
   Perf/boost/OC tables are parsed by host-CPU code (no GSP on P100) and fed to the
   PMU as runtime arguments — nothing signs or hashes them. A modified perf table is
   indistinguishable from a genuine one at the falcon.
   *(Details: `docs/01-falcon-pmu-analysis.md`.)*

3. **The actual performance caps are ordinary, unprotected data tables:**
   - **P0 boost ceiling = 1328.5–1328.6 MHz** — the NVIDIA *PCIe-bin* ceiling,
     151 MHz below the **1480 MHz** boost NVIDIA validated on the same GP100 16 GB
     HBM2 silicon in the SXM2 P100. Lives in the Virtual P-State Table (Q15.16 u32s)
     plus four u24 mirrors in the Performance Table block.
   - **Overclocking Table body = all zeros** — blocks any in-band clock offset
     (`nvidia-smi -lgc/-rmc` beyond stock rejected). This is the in-band half of the
     "hard stop".
   - **Power limit fixed at 250 W** (default = max = `0x0003D090` mW = 250,000 mW) —
     correct for the 250 W PCIe board; **keep**.
   - **Memory clock 715 MHz** (1430 Mbps HBM2) — the HBM2 part rating; **keep**.
   *(Details: `docs/02-perf-table-analysis.md`; full pointer map in
   `docs/03-bit-perf-ptrs.md` and `data/perf-ptrs.json`.)*

4. **Hardware-safe fix (proposed, not applied): raise the P0 boost ceiling only,
   to 1480 MHz** — the highest clock NVIDIA validated for this exact die — by
   patching **6 sites per ROM** (4× u24 mirrors + 2× VP-table Q15.16 records;
   `1480 × 128 = 0x02E400`, `1480 × 32768 = 0x02E40000`). Every protective
   mechanism stays untouched: 250 W power cap, power brake, thermal
   slowdown/shutdown, voltage/rail/AVFS tables, DI/DT, and the encrypted PMU
   partition. Because Boost 3.0 *requests* clocks while AVFS and the power brake
   *grant* them, the change is self-limiting: a weak or hot card simply never
   reaches 1480 MHz and the card can never exceed its 250 W design TDP. A
   conservative 1450 MHz first stage is also specified. Exact byte patches with
   offsets and before/after values: `docs/02-perf-table-analysis.md` §8.

---

## Repository structure

```
.
├── README.md                        — this file
├── docs/
│   ├── 01-falcon-pmu-analysis.md    — the "falcon ucode table" verdict, ACR/PMU lock
│   │                                  model, and why perf tables are unprotected
│   ├── 02-perf-table-analysis.md    — full parse of Performance / Virtual P-State /
│   │                                  Memory Clock / OC / Power Policy tables, cap
│   │                                  analysis, and the hardware-safe patch spec
│   └── 03-bit-perf-ptrs.md          — BIT PERF_PTRS (token 0x50) pointer map, all cards
├── data/
│   ├── perf-ptrs.json               — raw BIT PERF_PTRS pointer data (4 cards)
│   └── SHA256SUMS                   — checksums of the analyzed ROM files
├── references/
│   └── nouveau-reference.md         — which open-source driver files were used and
│                                      what each one shows
└── LICENSE                          — MIT
```

## Reading order

1. This README.
2. `docs/02-perf-table-analysis.md` — where the caps are and how to lift them safely.
3. `docs/01-falcon-pmu-analysis.md` — why the only real "hard stop" is the encrypted
   PMU partition, which this fix deliberately does **not** touch.
4. `docs/03-bit-perf-ptrs.md` / `data/perf-ptrs.json` — the pointer map used to
   locate everything.

## License

MIT — see `LICENSE`.
