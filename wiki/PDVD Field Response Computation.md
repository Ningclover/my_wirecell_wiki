---
tags: [algorithm, concept, experiment]
sources: 1
updated: 2026-05-03
---

# PDVD Field Response Computation

Step-by-step procedure for computing the ProtoDUNE-VD (3-view PCB strip) field response functions using the `pochoir` toolkit. Produces a WireCell-format FR JSON file (`FR_xn_new.json.bz2`) consumed by the signal processing chain.

## Purpose

The field response (FR) encodes how a drifting point charge induces current on each wire plane as a function of its transverse position within a strip pitch. WireCell uses these tabulated waveforms in deconvolution. The PDVD geometry has three views: U and V (30° induction strips) and W (90° collection strips), all on a PCB with circular holes.

## Pipeline Overview

The computation is implemented in `pochoir` (Python), run from `test/test-full-3d-drift-3view_all_xn.sh`. The store (output directory) is set via `POCHOIR_STORE` (e.g. `store_new_multi/`). A `want <key> <cmd>` idiom skips steps already completed.

### Stage 1 — Domain Definition

`pochoir domain` defines a finite uniform grid (shape × spacing) for each simulation sub-problem:

| Key | Shape | Spacing | Purpose |
|-----|-------|---------|---------|
| `domain/drift3d` | 25×15×2100 | 0.1 mm | 3D drift field (quarter-cell symmetry) |
| `domain/weight2d_u` | 1607×2100 | 0.1 mm | 2D weighting potential, U plane |
| `domain/weight3d_u` | 536×30×2100 | 0.1 mm | 3D weighting potential, U plane (7 strips) |
| `domain/weight2d_v` | 1607×2100 | 0.1 mm | 2D weighting potential, V plane |
| `domain/weight3d_v` | 536×30×2100 | 0.1 mm | 3D weighting potential, V plane (7 strips) |
| `domain/weight2d_w` | 1071×2100 | 0.1 mm | 2D weighting potential, W plane |
| `domain/weight3d_w` | 357×30×2100 | 0.1 mm | 3D weighting potential, W plane (7 strips) |

Z-axis (2100 points × 0.1 mm = 210 mm) is the drift direction. X is the strip-pitch direction; Y is along the strip.

### Stage 2 — Initial & Boundary Value Generation

`pochoir gen` paints electrode and PCB boundary conditions onto each domain using PCB geometry generators:

- **`drift3d`**: generator `pcb_quarter_30deg`, config `example_gen_pcb_quarter_config_30deg.json`
  - PCB hole radii: 1.2 mm (both), PCB width: 3.2 mm
  - Potentials: Collection +1000 V, Induction1 −500 V, Induction2 0 V, Cathode −9995 V
  - Ground position: 7 mm; PCB low edge: 17 mm
- **`weight2d_u/v`**: generator `pcb_2D_30deg`, per-plane config (plane index 1 for U/plane 2 for V)
  - Strip pitch U/V: 7.65 mm, 21 strips; hole diameter 2.4 mm, between-holes gap 1.35 mm
- **`weight3d_u/v`**: generator `pcb_3D_30deg`, per-plane config
  - `QuarterDimX=2.55 mm`, `QuarterDimY=1.47 mm`, 7 strips
- **`weight2d_w` / `weight3d_w`**: generators `pcb_2D_90deg` / `pcb_3D_90deg`, config `example_gen_pcb_2D_config.json` / `example_gen_pcb_3D_config.json`
  - Strip pitch W: 5.1 mm, `QuarterDimX=2.55 mm` (same PCB cell), plane index 0

Output: `initial/<key>.npz` + `boundary/<key>.npz` in the store.

### Stage 3 — FDM Solve (2D potentials)

`pochoir fdm` runs the finite-difference method (Laplace equation, engine `cumba` = CUDA+Numba):

- **Drift & 2D weighting fields**: 20 epochs × 200,000 iterations, precision 5×10⁻⁸, all-`fix` edges
- Output: `potential/<key>.npz` + `increment/<key>.npz`

The drift3d FDM uses the quarter-cell geometry; the weight2D fields span the full 2D cross-section.

### Stage 4 — 3D Boundary Condition Interpolation

`pochoir bc-interp` stitches the 2D solution into the boundary of the 3D weighting problem:

- Reads: `potential/weight2d_<p>`, `initial/weight3d_<p>`, `boundary/weight3d_<p>`
- `--xcoord`: sets the x-distance from center at which the 2D BC is applied
  - U/V: `26.78 mm` = 3 × QuarterDimX (= 3 × 2.55 mm) × ... (number of active sensing strips × stripX / 2)
  - W: `17.85 mm` = 2 × QuarterDimX × ...
- Output: `initial/weight3dfull_<p>.npz` + `boundary/weight3dfull_<p>.npz`

### Stage 5 — FDM Solve (3D weighting potentials)

`pochoir fdm` on each `weight3dfull_<p>`: 10 epochs × 9,000 iterations, precision 1×10⁻⁵, all-`fix` edges.

Output: `potential/weight3dfull_<p>.npz`

### Stage 6 — Extend Weighting Field to Full Volume

`pochoir extendwf` merges the 3D near-strip region with the 2D far-field solution to produce a full-volume weighting potential:

- Central 7-strip block (`i < 7×onestrip`): use 2D solution for all Y slices
- Next 7-strip block (strip indices 7–14): use 3D solution for Z < 1100 (near PCB), 2D for Z ≥ 1100
- Beyond 14-strip block: use 2D solution
- `onestrip = domain3d.shape[0] / 7`
- Output: `potential/weight3dextend_<p>.npz` (shape = `sol2D.shape[0]` × `dom3D.shape[1]` × `dom2D.shape[1]`)

### Stage 7 — Drift Velocity Field

`pochoir velo` converts the drift3d potential into a 3-component velocity field at T = 89 K (LAr):

- Computes E-field gradient from potential, zeroes boundary-adjacent cells
- Applies LAr mobility model (`pochoir.lar.mobility`) to map |E| → µ
- Velocity = µ × E (units: mm/µs)
- Output: `velocity/drift3d.npz`

### Stage 8 — Drift Path Tracing

`pochoir starts` + `pochoir drift` trace electron trajectories through the velocity field:

Two sets of start points, all at Z = 198 mm (just inside the sensitive volume):

- **U/V paths** (`starts/drift3d`): 120 points on a 6-X × 10-Y grid
  - X positions (mm): 0.05, 0.52, 0.81, 1.28, 1.56, 2.32 — cover one 30° strip unit cell
  - Y positions (mm): 0.05 to 1.31 in steps of 0.14 (10 points)
  - Paths integrated over `0–300 µs` in 0.1 µs steps
  - Output: `paths/drift3d.npz`
- **W paths** (`starts/drift3d_w`): 60 points on a 6-X × 10-Y grid
  - X positions (mm): 0.05, 0.51, 1.02, 1.53, 2.04, 2.54 — cover one 90° strip unit cell
  - Paths: same time range and step
  - Output: `paths/drift3d_w.npz`

### Stage 9 — Induced Current Calculation

`pochoir induce-30deg` (U/V) and `pochoir induce` (W) compute the current waveform induced on each strip as paths traverse the extended weighting field:

- Evaluates `Q(t) = charge × W(path(t))` via 3D interpolation (RegularGridInterpolator)
- `I(t) = dQ/dt`
- `--nstrips 21`: mirrors paths across 21 strips, shifting X by `dx = 7.615 mm` (U/V) or `5.1 mm` (W) per strip
- `--average 10`: averages 10 paths per output waveform (averaging over Y positions within a strip)
- The 30-deg variant (`induce-30deg`) handles the two orientations of the 30° strip unit cell (odd/even strip index alternates the Y offset by ±1.45 mm)
- `-S paths/weight3dextend_<p>`: also saves the shifted path array to the store
- Output: `current/induced_current_avg_ind_<p>.npz` (shape: `21 × 6 × nsteps`)

### Stage 10 — WireCell Format Conversion

`pochoir convertfr` packages all three plane currents into a WireCell-format FR file:

Config: `example_convertfr_vd.json`:
- `origin=181.0 mm`, `speed=0.00153 mm/ns` (drift speed), `tstart=0`, `period=100 ns`
- `totstrip=21`, `npaths=6`
- Pitches: U=7.65 mm, V=7.65 mm, W=5.1 mm
- Path spacing within strip: U=0.765 mm, V=0.765 mm, W=0.51 mm
- Plane locations: U=13.2 mm, V=3.2 mm, W=0.0 mm (relative to collection)

Procedure:
- Each strip contributes `npaths=6` `PathResponse` entries starting at `−pitch×nstrips/2`
- Pitch positions advance by `in_strip_shift` between paths, then by `between_strip_shift` between strips
- Truncates each waveform to 1325 samples and flips sign (`-1 × current`)
- Builds `PlaneResponse` → `FieldResponse` schema objects and serializes to bz2 JSON

Output: `current/FR_xn_new.json.bz2`

## Key Configuration Files

| File | Purpose |
|------|---------|
| `example_gen_pcb_quarter_config_30deg.json` | Drift domain geometry + potentials |
| `example_gen_pcb_2D_config_30deg_u.json` | U-plane 2D strip geometry |
| `example_gen_pcb_2D_config_30deg_v.json` | V-plane 2D strip geometry |
| `example_gen_pcb_2D_config.json` | W-plane 2D strip geometry |
| `example_gen_pcb_3D_config_30deg_u.json` | U-plane 3D strip (7 strips, QuarterDimX=2.55 mm) |
| `example_gen_pcb_3D_config_30deg_v.json` | V-plane 3D strip (7 strips) |
| `example_gen_pcb_3D_config.json` | W-plane 3D strip |
| `example_convertfr_vd.json` | WireCell FR output parameters |
| `helpers.sh` | `want` / `want_file` skip-if-exists guards |

## Output Store Layout

All intermediate arrays stored in `store_new_multi/` (or `POCHOIR_STORE`):

```
store_new_multi/
  domain/         — grid definitions (.npz + .json)
  initial/        — initial value arrays per domain
  boundary/       — boundary mask arrays per domain
  potential/      — FDM solutions (drift3d, weight2d/3d/3dfull/3dextend per plane)
  increment/      — FDM residuals
  velocity/       — drift velocity field
  starts/         — electron start point arrays
  paths/          — drift trajectories
  current/        — induced current waveforms + FR_xn_new.json.bz2
```

## Data Flow DAG

```
domain/* ──► gen (initial/*, boundary/*) ──► fdm (potential/weight2d_*)
                                              │
domain/drift3d ──► gen ──► fdm ─────────────► velo ──► drift ──► paths/*
                                              │
                          potential/weight2d_* ──► bc-interp ──► fdm ──► potential/weight3dfull_*
                                                                          │
                                                                     extendwf ──► potential/weight3dextend_*
                                                                          │
                                                               paths/* + potential/weight3dextend_* ──► induce ──► current/*
                                                                          │
                                                                     convertfr ──► FR_xn_new.json.bz2
```

## Key Implementation Notes

- `pochoir fdm` uses the `cumba` engine (CUDA + Numba) for GPU-accelerated Laplace solving.
- The `want` guard checks for `$POCHOIR_STORE/<key>.npz` before running a step — re-running the script is idempotent.
- `bc-interp` xcoord formula: `(N_active_strips × stripX) / 2`, where `stripX = 3×QuarterDimX` for U/V and `2×QuarterDimX` for W.
- `extendwf` uses a hardcoded `cut_z=1100` grid point to switch from 3D to 2D solution along the drift axis.
- The 30° strip geometry requires two orientations of the unit cell; `induce-30deg` alternates the Y shift (±1.45 mm) between even/odd strips.
- `convertfr` truncates FR waveforms to **1325 samples** (132.5 µs at 100 ns period).
- The output FR file is the input to `ctoffset` tuning in `sp.jsonnet` — the `ctoffset = 4 µs` is set to match this FR.

## Source Files

| File | Role |
|------|------|
| `test/test-full-3d-drift-3view_all_xn.sh` | Master run script |
| `test/helpers.sh` | `want` / `want_file` helpers |
| `pochoir/__main__.py` | All CLI commands: `domain`, `gen`, `fdm`, `velo`, `starts`, `drift`, `bc-interp`, `extendwf`, `induce`, `induce-30deg`, `convertfr` |
| `pochoir/main.py` | `Main` class: store I/O, `put_domain`, `get_domain`, `put`, `get` |
| `pochoir/gen_pcb_quarter_30deg.py` | Drift domain geometry generator |
| `pochoir/gen_pcb_2Dstrips_30deg.py` | 2D strip generator (30°) |
| `pochoir/gen_pcb_3Dstrips_30deg.py` | 3D strip generator (30°) |
| `pochoir/gen_pcb_2Dstrips_90deg.py` | 2D strip generator (90°/W) |
| `pochoir/gen_pcb_3Dstrips_90deg.py` | 3D strip generator (90°/W) |
| `pochoir/fdm_cumba.py` | CUDA+Numba FDM engine |
| `pochoir/bc_interp.py` | 2D→3D BC interpolation |
| `pochoir/drift_numpy.py` | Electron drift path integration |
| `pochoir/lar.py` | LAr mobility / diffusion models |
| `pochoir/schema.py` | `PathResponse`, `PlaneResponse`, `FieldResponse` WireCell schema |
| `pochoir/persist.py` | Store I/O (`dumpfr`, npz/json backends) |

## See also

[[PDVD Detector Parameters]], [[PDVD Signal Processing Configuration]], [[ADC to Electrons Signal Chain]]

## Sources

- [[source-session-2026-05-03-pdvd-field-response]]
