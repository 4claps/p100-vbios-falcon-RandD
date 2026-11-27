# Open-source driver reference material

The analysis was cross-checked against source files from two open-source NVIDIA
driver codebases, both MIT-licensed upstream:

- **nouveau** — the open-source NVIDIA kernel driver in the Linux tree
  (`drivers/gpu/drm/nouveau/`).
- **open-gpu-kernel-modules** (ogkm) — NVIDIA's open kernel modules.

Files were kept locally as read-only reference (not redistributed in this repo).
This page summarizes what each file is and which claim it supported.

## Chip wiring and device model

| File | Origin | What it shows |
|---|---|---|
| `dev_base.c` | nouveau `nvkm/core` | Master chipset table mapping each NVIDIA chip to its subdevice constructors. The **`nv130_chipset` (GP100)** entry is the key evidence: `pmu = gm200_pmu_new`, `acr = gm200_acr_new`, `fb/mc/ltc/... = gp100_*`. This proves GP100 reuses the GM200-generation PMU and ACR implementations. |
| `nouveau_files.txt` | nouveau tree listing | Full 1,236-entry file manifest of the nouveau driver tree, used for locating the relevant subdevice sources. |

## vBIOS parsing

| File | Origin | What it shows |
|---|---|---|
| `bios_base.c` | nouveau `subdev/bios` | Generic PCI expansion-ROM image parsing (PNP/PCIR headers, image enumeration) — the model used to map image offsets vs file offsets (`file = image + 0xA00` in these dumps). |
| `bios_image.c` | nouveau `subdev/bios` | PCI image header (`PCIR`) parsing: vendor/device/class/revision fields — used to identify image #2 (device `0x15f8`, rev `0x85`) in the ROM. |
| `bit.c` | nouveau `subdev/bios` | **BIT table walk**: entry layout (id/version/length/offset records, stride from table header) — the algorithm used to locate the BIT and enumerate its tokens, including token `0x50` (PERF_PTRS). Confirms the BIT on gpu0 (`0xbb0`) is the PREOS code-vector table. |
| `bios_pmu.c` | nouveau `subdev/bios` | Helper to read the PMU-related BIT pointer (token `'P'`) from the vBIOS — context for how the driver itself discovers PMU tables. |

## Falcon / PMU

| File | Origin | What it shows |
|---|---|---|
| `falcon_base.c` | nouveau `nvkm/core` | Falcon core lifecycle: halt/reset, ucode load path, size-limit checks (`ucode exceeds falcon limit(s)`), signature handling hooks — the general falcon engine model. |
| `pmu_base.c` | nouveau `subdev/pmu` | PMU subdevice core: firmware interface selection (`fwif`), msgq/cmdq RPC plumbing — how the OS driver talks to an already-running PMU. |
| `pmu_gm200.c` | nouveau `subdev/pmu` | **Key evidence:** the GM200 PMU `fwif` table contains only `{ -1, gm200_pmu_nofw, ... }` — `gm200_pmu_nofw` means "firmware unavailable": **the open driver never loads PMU ucode**; the PMU firmware is already running (booted by vBIOS PREOS during POST). |
| `pmu_gp102.c` | nouveau `subdev/pmu` | GP102+ PMU variant (reset/enabled register handling, `flcn = gm200_pmu_flcn`); companion to the GP100/nv130 wiring in `dev_base.c`. |
| `rm_kern_pmu.c` | ogkm `src/nvidia/src/kernel/gpu/pmu` | NVIDIA RM's own PMU kernel object for the same architectures — cross-check that even the proprietary stack on these chips interacts with the PMU via RPC rather than loading its ucode at OS time. |
| `rm_kernel_falcon.c` | ogkm `src/nvidia/src/kernel/gpu/falcon` | NVIDIA RM's falcon core implementation — engine lifecycle, secure vs non-secure attach — used to confirm the falcon boot/validation model matches nouveau's. |

## ACR (Authenticated Code Region) — the actual lock

| File | Origin | What it shows |
|---|---|---|
| `acr_base.c` | nouveau `subdev/acr` | ACR subdevice core: LS (light-segment) firmware management, WPR handling — the architecture of the "lock" that protects falcon firmware. |
| `acr_gm200.c` | nouveau `subdev/acr` | GM200/GP1xx ACR implementation: HS/FWSEC bootstrap, WPR protection of the LS images — the exact variant GP100 uses per `dev_base.c`. |
| `acr_gp102.c` | nouveau `subdev/acr` | GP102+ ACR variant (revocation/attach details) — companion evidence for the GP1xx family. |
| `acr_hsfw.c` | nouveau `subdev/acr` | HS/FWSEC image loading: FWSECLIC binary loading and attachment to the WPR. |
| `acr_lsfw.c` | nouveau `subdev/acr` | LS firmware load path: `sig` + `image` file pairs loaded per LS (e.g. PMU), signature blob handling — shows the **ucode images themselves are signed**, and ACR validates them. |
| `nvfw_acr.c` / `nvfw_acr.h` | nouveau `nvfw` | NVFW firmware format helpers + `wpr_header` v0/v1 struct definitions with status codes (`VALIDATION_CODE_FAILED`, `VALIDATION_DATA_FAILED`, `REVOCATION_CHECK_FAILED`, `BOOTSTRAP_READY`) — the validation-state machine that produces a dead card if the firmware region is tampered with. |
| `nvfw_ls.c` / `nvfw_ls.h` | nouveau `nvfw` | LS image descriptor/header parsing (`nvfw_ls_desc_head`, `nvfw_bin_hdr`) — the signed-image container format. |
| `nvfw_fw.h` | nouveau `nvfw` | Base NVFW binary header struct (`bin_magic`, `bin_ver`) — used in the magic-scan hunt for ucode signatures in the ROM. |
| `acr_h.h` / `acr_priv.h` | nouveau `subdev/acr` | ACR subsystem/priv headers: LS IDs (`NVKM_ACR_LSF_PMU`), fwif structs — structural context. |

## What this material established

1. GP100's PMU firmware is **booted by vBIOS PREOS code at POST**, not by the OS
   driver (`pmu_gm200.c` `nofw`, `dev_base.c` wiring).
2. ACR protects **only the falcon firmware images** — WPR + RSA-2048-signed
   HS/LS images (`acr_*.c`, `nvfw_*`): tampering in the encrypted partition fails
   validation → card does not boot.
3. **No public spec or parser exists for the GP100 perf-table block**, and neither
   nouveau nor ogkm validate or sign those tables — which is why the boost/OC
   tables (see `docs/02-perf-table-analysis.md`) are plain, editable data.

*The upstream files remain under their original MIT licenses (Copyright Red Hat
Inc. 2012–2019 for nouveau files; Copyright NVIDIA Corporation 2021–2023 for ogkm
files). They are intentionally not copied into this repo.*
