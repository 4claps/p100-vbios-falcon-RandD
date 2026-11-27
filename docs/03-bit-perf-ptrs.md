# BIT PERF_PTRS table reference (token 0x50)

The **PERF_PTRS** BIT table (BIT token/ID `'P'` = `0x50`) is the master pointer
index that locates every performance/power/thermal sub-table in the vBIOS.

**Source note:** BIT PERF_PTRS v2 (token 0x50). Pointers are u32 offsets from the PCI-expansion-ROM image base (file offset 0xa00 in these dumps; strings/version cross-verified). Table=156B=39 pointers; names 1-39 of spec order; 40th (LowPower NVLINK Table) has no slot.

**Offset convention:** pointers below are **u32 image offsets** (from the
PCI expansion-ROM image base). In the extracted dumps analyzed here,
**file offset = image offset + 0xA00**.

**BIT location itself:** on gpu0 the PREOS code-vector BIT lives at file `0xbb0`;
the PERF_PTRS table referenced here was located via the token-`0x50` entry
(pointer to table data: gpu0 image offset `702` = `0x2be`,
gpu1–gpu3 `750` = `0x2ee`).

Table layout: 156 B = 39 u32 pointers, in the spec name order below (entries 1–39;
the 40th spec table, *LowPower NVLINK Table*, has no slot in this version).
`0x00000` = table absent.

| # | Table name | gpu0 img | gpu1/2/3 img | gpu0 file | gpu1/2/3 file | identical img? |
|---|---|---|---|---|---|---|
| 1 | Performance Table | `0x0b06c` | `0x0b05a` | 0x0ba6c | 0x0ba5a | no |
| 2 | Memory Clock Table | `0x0b340` | `0x0b32c` | 0x0bd40 | 0x0bd2c | no |
| 3 | Memory Tweak Table | `0x00000` | `0x00000` | — | — | yes |
| 4 | Power Control Table | `0x00000` | `0x00000` | — | — | yes |
| 5 | Thermal Control Table | `0x00000` | `0x00000` | — | — | yes |
| 6 | Thermal Device Table | `0x0ca8f` | `0x0c9da` | 0x0d48f | 0x0d3da | no |
| 7 | Thermal Coolers Table | `0x00000` | `0x00000` | — | — | yes |
| 8 | Performance Settings Script | `0x00000` | `0x00000` | — | — | yes |
| 9 | Continuous Virtual Binning Table | `0x0b546` | `0x0b532` | 0x0bf46 | 0x0bf32 | no |
| 10 | Ventura Table | `0x00000` | `0x00000` | — | — | yes |
| 11 | Power Sensors Table | `0x0bf75` | `0x0bf61` | 0x0c975 | 0x0c961 | no |
| 12 | Power Policy Table | `0x0c363` | `0x0c34f` | 0x0cd63 | 0x0cd4f | no |
| 13 | P-State Clock Range Table | `0x00000` | `0x00000` | — | — | yes |
| 14 | Voltage Frequency Table | `0x00000` | `0x00000` | — | — | yes |
| 15 | Virtual P-State Table | `0x0b172` | `0x0b160` | 0x0bb72 | 0x0bb60 | no |
| 16 | Power Topology Table | `0x0c010` | `0x0bffc` | 0x0ca10 | 0x0c9fc | no |
| 17 | Power Leakage Table | `0x0c901` | `0x0c84c` | 0x0d301 | 0x0d24c | no |
| 18 | Performance Test Specifications Table | `0x0b31e` | `0x0b30a` | 0x0bd1e | 0x0bd0a | no |
| 19 | Thermal Channel Table | `0x0ca9f` | `0x0c9ea` | 0x0d49f | 0x0d3ea | no |
| 20 | Thermal Adjustment Table | `0x0cad0` | `0x0ca1b` | 0x0d4d0 | 0x0d41b | no |
| 21 | Thermal Policy Table | `0x0ca19` | `0x0c964` | 0x0d419 | 0x0d364 | no |
| 22 | P-State Memory Clock Frequency Table | `0x00000` | `0x00000` | — | — | yes |
| 23 | Fan Cooler Table | `0x0cade` | `0x0ca29` | 0x0d4de | 0x0d429 | no |
| 24 | Fan Policy Table | `0x0cafc` | `0x0ca47` | 0x0d4fc | 0x0d447 | no |
| 25 | DI/DT Table | `0x0cb59` | `0x0caa4` | 0x0d559 | 0x0d4a4 | no |
| 26 | Fan Test Table | `0x0cb36` | `0x0ca81` | 0x0d536 | 0x0d481 | no |
| 27 | Voltage Rail Table | `0x04f1e` | `0x04fa0` | 0x0591e | 0x059a0 | no |
| 28 | Voltage Device Table | `0x04eba` | `0x04f3c` | 0x058ba | 0x0593c | no |
| 29 | Voltage Policy Table | `0x04f39` | `0x04fbb` | 0x05939 | 0x059bb | no |
| 30 | LowPower Table | `0x0b52a` | `0x0b516` | 0x0bf2a | 0x0bf16 | no |
| 31 | LowPower PCIe Table | `0x0b535` | `0x0b521` | 0x0bf35 | 0x0bf21 | no |
| 32 | LowPower PCIe-Platform Table | `0x00000` | `0x00000` | — | — | yes |
| 33 | LowPower GR Table | `0x00000` | `0x00000` | — | — | yes |
| 34 | LowPower MS Table | `0x00000` | `0x00000` | — | — | yes |
| 35 | LowPower DI Table | `0x00000` | `0x00000` | — | — | yes |
| 36 | LowPower GC6 Table | `0x00000` | `0x00000` | — | — | yes |
| 37 | LowPower PSI Table | `0x00000` | `0x00000` | — | — | yes |
| 38 | Thermal Monitor Table | `0x00000` | `0x00000` | — | — | yes |
| 39 | Overclocking Table | `0x0cb3f` | `0x0ca8a` | 0x0d53f | 0x0d48a | no |

## Notes

- **gpu1, gpu2 and gpu3 are byte-identical for every pointer** (same vBIOS build,
  PCIR rev `0x84`), so they share one column.
- gpu0 (rev `0x85`) generally carries its perf-span tables **0x12 (18) bytes later**
  than gpu1 — a consequence of gpu0's 2-byte-longer Virtual P-State header — while
  its BIT lives 0x30 earlier (see `02-perf-table-analysis.md` §5).
- The voltage tables (`Voltage Rail/Device/Policy`, entries 27–29) live well below
  the perf span; they are intentionally untouched by the proposed fix.
- Absent tables (`0x00000`) on this board: Memory Tweak, Power Control, Thermal
  Control, Thermal Coolers, Performance Settings Script, Ventura, P-State Clock
  Range, Voltage Frequency, P-State Memclk Freq, most LowPower sub-tables, Thermal
  Monitor.
- Raw machine-readable data: [`../data/perf-ptrs.json`](../data/perf-ptrs.json).
