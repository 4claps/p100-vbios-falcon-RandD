# Related work: FBR RAM Downloader (V100 live memory-clock unlock)

https://github.com/FBR-ROM/FBR-RAM-Downloader — researched 2026-11-28. Not a GP100
tool, but the closest published sibling of the problem analysed in this repo: the
same "sibling SKU validated faster clocks on identical silicon" lock, attacked at
the register level instead of the firmware level. MIT licensed, v0.1.2.

## What it does

Raises Tesla V100 **HBM2 memory clocks live** — 877.5 → 1053 MHz (daily, +20 %) or
1080 MHz (sprint) — with no VBIOS flash, no driver reload, no reboot. Volatile: a
reboot (or `restore`) returns stock clocks. Measured by the authors: memtest-clean
at 1053, byte-identical serving output, +10.1 % decode throughput. **1107 MHz (the
V100S bin) failed immediately** — binned-up clocks are not guaranteed on real cards.

## Mechanism (GV100)

- `mmap` of `/sys/bus/pci/devices/<bdf>/resource0` (BAR0) as root; a single write
  of the HBM PLL coefficient to the **write-only multicast register `0x98BC98`**
  reprograms all 16 memory partitions (FBPAs) at once.
- Clock = `27 MHz × NDIV / (MDIV × PLDIV)`; stock NDIV=65 → 78 daily → 80 sprint
  (MDIV=2, PLDIV=1 fixed).
- **Reading the multicast window `0x988000–0x98C000` hangs the GV100 PRI register
  bus** — the tool refuses reads there at two independent code layers.
- Verification uses hardware ground truth, not the driver: free-running 36-bit
  DRAM cycle counters per HBM PLL (3 samples/PLL over a 500 ms window, hi/lo
  wrap-safe) must land within ±1 MHz of target. `nvidia-smi` keeps reporting the
  stock max after a live change.
- `PMC_BOOT_0` chip-ID canary read before *and* after every register step; all-ones
  and `0xBADxxxxx` treated as bus errors.

## Safety model worth copying

- Refuse to run from an unknown state: stock PLL dividers required, current clock
  must be inside `stock..target`, else "reboot to reset".
- All-or-nothing across cards: every card surveyed before anything is written; any
  failure or user cancellation rolls every touched card back to its original
  coefficient; a failed write-back is an explicit "stranded — power-cycle the host"
  state and that card is never touched again.
- Dry-run survey mode first; distinct exit codes per outcome; a 3,000-run seeded
  chaos soak against a simulator with random faults and mid-run cancellation.
- Lineage from the mining-card unlock scene (duggasco/CMP100-210,
  cachenetics/170tune); honest failure disclosure throughout (their first method
  crashed the card/host 5 of 7 times; the README leads with an ECC warning).

## Relation to the GP100 "vBIOS hard stop"

- **Different lock:** the V100 memory cap is enforced in the driver (single
  exposed speed) over a claimed signed firmware; the GP100 core-clock cap is a
  plain unsigned *data table* in the vBIOS parsed by host code (see
  `docs/01-falcon-pmu-analysis.md`) — which is why the GP100 fix here is a ROM
  patch, not a register poke.
- **Different axis:** the P100's HBM2 runs at 715 MHz on *both* PCIe and SXM2
  SKUs — the part rating — so the V100 memory-bandwidth win has no P100 analog.
  The GP100 headroom is core clock (1329 → 1480 MHz).
- **Conceptually transferable, unverified:** a no-flash P100 core-clock unlock via
  BAR0 writes to the GP100 GPC PLL registers. Caveat: GP100 core clocks are
  actively DVFS'd by the PMU (the V100 memory PLL is never touched at runtime), so
  a one-shot register write would likely race or be reprogrammed at the next boost
  event. Chasing it would need fresh REA work on GP100 clock registers.
- The *discipline* transfers directly to `docs/04-step-by-step-flash-guide.md`:
  canary/health checks around every step, hardware-counter-level verification
  instead of trusting `nvidia-smi`, per-card all-or-nothing with rollback,
  refuse-to-run state gating, and an explicit unrecoverable-state policy.
