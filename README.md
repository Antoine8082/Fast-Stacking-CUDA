# FS-CUDA — Fast Stacking CUDA

Clean-room, GPU-accelerated astrophotography stacker for one-shot-colour (OSC)
data. A standalone Windows application with no third-party astro-platform
dependency.

**Goal:** the fastest possible processing at the highest possible image
quality — from raw light frames to a finished linear master.

> Proprietary donationware (not open source). Distributed as `FS-CUDA.exe`;
> see [EULA.txt](EULA.txt) and [docs/DISTRIBUTION.md](docs/DISTRIBUTION.md). This
> repository is the private development tree; the public release carries only
> the binary and a user readme.

## v2 fused engine

Since v2, stacks of ≥128 frames run on a **fused GPU pipeline**: raw 16-bit
CFA frames stay RAM-resident (18 MB/frame instead of 108 MB float RGB), are
read from disk exactly once, and calibration → cosmetic → debayer → warp →
rejection all execute on the GPU per horizontal band — only finished master
rows return to the host. Rejection uses an exact radix-select k-th-order
kernel (measured fastest against sort- and CPU-based alternatives at every
VRAM-fit configuration). Small stacks (<128 frames) keep the v1 in-memory
path, which is slightly faster there; both engines produce byte-identical
masters.

Measured, 1089 × 3008×3008 real frames (RTX 5070 laptop, 64 GB RAM):

| engine | wall time | GPU | notes |
|---|---|---|---|
| v1 streaming | 414 s | bursty | disk spill, CPU rejection |
| **v2 fused** | **158 s** | 100 % sustained | zero spill, ~6.7 GB VRAM, identical master |

Trail rejection used to fall off this engine entirely — the detector needs a
whole registered frame and the fused pipeline never builds one — so ticking the
box silently bought you the v1 engine. Since 1.11 it detects in a GPU pre-pass
and masks per band instead. Measured, 300 × 3008×3008 real frames:

| run | v1 engine | fused | master |
|---|---|---|---|
| trails | 75.7 s | **35.3 s** | byte-identical |
| trails + local normalization | 93.7 s | **50.3 s** | byte-identical |
| no trails (reference point) | — | 25.0 s | — |

Nearly every option now runs fused: per-pixel inverse-variance weighting,
autocrop, local normalization, crash-resume and the RCD debayer (the
default — it roughly halves round-star chroma ringing) are all
byte-exact with the v1 engine — as is trail rejection, in any combination.
Defect repair, drizzle in every form, and the GESD and linear-fit
rejection modes run fused too. **SuperPixel is the only option left that
routes to v1**, and `fused_v1_reasons()` is the single source of truth for
that — the engine gate, the GUI warning and a test all read it. Drizzle
runs a dedicated quality pipeline: per-pixel sigma-clip rejection,
satellite-trail masks, normalization and autocrop are applied to every
drop (legacy ungated drizzle remains available by turning those options
off). No option has a RAM ceiling — anything too
large to hold in memory streams from disk. `FS_FUSED=0` forces v1;
`FS_FUSED=1` forces fused at any frame count. Hard limit: 10 000 frames.

## Pipeline (OSC)

Input: a folder of raw light frames + a master dark + a master flat.
Output: one master image (32-bit float, linear, full metadata preserved).

1. **Read FITS** — 16/32-bit, all acquisition metadata kept.
2. **Calibrate** — dark subtract (optional dark scaling), flat divide.
3. **Cosmetic correction** — hot/cold pixel repair (optional).
4. **Linear defect correction** — column/row defects (optional).
5. **Debayer** — RCD (default; smooth gradient-weighted green, rings less on
   star cores) or SuperPixel (one RGB pixel per 2x2 cell, no interpolation —
   measured 36% deeper on faint extended detail at matched scale). Mono sensors
   skip this entirely and stay single-channel.
6. **Measure** — FWHM, eccentricity, SNR / PSF weight, noise per frame.
7. **Frame selection** — interactive table + charts; keep/reject.
8. **Register** — star detect → triangle match → RANSAC affine →
   **sub-pixel refinement** → warp; optional thin-plate-spline distortion
   correction.
9. **Local normalization** — sky-gradient model with star-core protection.
10. **Integration** — **inverse-variance weighted**, robust rejection
    (winsorized sigma / GESD) with dedicated **satellite-trail rejection**.
11. **Drizzle / Bayer drizzle** — optional (undersampled, well-dithered sets).
12. **Autocrop** — optional.
13. **Plate solve** — built-in Gaia DR3 solver (optional; WCS in header).
14. **RGB alignment** — optional.
15. **Write FITS master** — 32-bit float, linear, metadata + processing history.

## Architecture

- `core/` — C++/CUDA stacking engine (CPU fallback). Pure compute; no UI or
  platform dependency.
- `io/` — FITS read/write, metadata-preserving.
- `app/` — dead-simple native Windows GUI (Dear ImGui + GLFW), including the
  interactive frame-selection view.
- `tests/` — CPU-vs-CUDA oracle checks + known-answer tests.

## Principles

- **32-bit float, linear light, end to end.** No hidden precision loss.
- **Bounded memory.** Frames stream through; RAM and VRAM stay flat no matter
  how many frames are stacked.
- **Correct before clever.** Inverse-variance weighting and sub-pixel
  registration are the two levers that decide final quality — both are built
  in from the start.
- **Clean-room.** Every algorithm is implemented from published literature
  (cited in the code). No code is derived from any third-party astro platform.

## Build

Windows x64, CUDA (optional — CPU fallback builds without it), CMake ≥ 3.18,
C++17.

```powershell
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

### Run

`fs_stack` stacks a folder of raw CFA lights into a linear RGB master;
`fs_synth` writes a synthetic test set to try it on.

```powershell
build\Release\fs_synth.exe data --frames 10
build\Release\fs_stack.exe data\lights --dark data\dark.fits --out master.fits
```

`fs_stack <lights-dir | light.fits...> [--dark F] [--flat F] [--out F]
[--pattern RGGB] [--method rcd|ha|superpixel] [--weight noise|psf]
[--detect-sigma X] [--no-cosmetic]`

### GUI

A Dear ImGui + GLFW front end (`fs_gui`) is opt-in — it fetches ImGui and GLFW
at configure time:

```powershell
cmake -B build -DCMAKE_BUILD_TYPE=Release -DFS_BUILD_GUI=ON
cmake --build build --config Release --target fs_gui
```

Pick a lights folder (plus optional master dark/flat), press Stack, and view
the linear master with an interactive asinh screen stretch. The pipeline runs
on a worker thread so the UI stays responsive.

## Status

Pipeline runs end to end: FITS I/O, calibration (dark/flat + CFA-aware
cosmetic), debayer (SuperPixel / RCD), per-frame measurement
(noise, star detection, FWHM/eccentricity/SNR), registration (triangle-match
star correspondence → RANSAC affine → 2nd-order distortion → bicubic warp),
and inverse-variance weighted, winsorized-sigma integration. `stack()` ties
them together: raw CFA lights (+ master dark/flat) → one linear RGB master.
All stages are covered by known-answer tests (`fs_tests`).

**GPU acceleration**: calibrate, debayer, and integrate run on CUDA when a
device is present, each validated bit-for-bit against the CPU path by an
oracle test (`--fmad=false`). The build falls back to a CPU-only path with no
toolkit. Calibrate+debayer are fused and resident (dark/flat uploaded once);
integrate is tiled to keep VRAM bounded. On an RTX 5070 a 100-frame 3008×3008
stack runs in ~13 s (≈15× the original single-threaded CPU time).

Validated on real IMX533 OSC data: 100 lights + 72 darks + 41 flats → a deep
linear master, noise reduced ~10× (√100), stars sharp.

**Optional stages** (off by default unless noted): linear column/row defect
correction, automatic frame selection, sub-pixel registration refinement (on),
2nd-order distortion correction, local (polynomial) background normalization,
GESD rejection, per-pixel inverse-variance weighting, satellite/plane trail
rejection, autocrop, RGB channel alignment, drizzle, and built-in Gaia plate
solving (Sesame target resolve + Gaia DR3 cone fetch, or a local Gaia CSV;
no external solver needed). Each is a checkbox/flag in the GUI
and `fs_stack`.

**GUI**: `fs_gui` (Dear ImGui + GLFW, Windows 11 theme) — pick folders, stack
on a worker thread, view the master with an interactive STF stretch. Build with
`-DFS_BUILD_GUI=ON`.
