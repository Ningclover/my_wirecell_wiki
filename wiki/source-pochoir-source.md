---
tags: [source]
type: file
date: 2026-05-07
context: Deep source code investigation of the pochoir Python package for LArTPC field response computation
files_touched:
  - pochoir/__main__.py
  - pochoir/fdm_generic.py
  - pochoir/fdm_numpy.py
  - pochoir/fdm_cumba.py
  - pochoir/domain.py
  - pochoir/lar.py
  - pochoir/drift_numpy.py
  - pochoir/bc_interp.py
  - pochoir/schema.py
  - pochoir/arrays.py
  - pochoir/gen.py
  - pochoir/gen_pcb_quarter_30deg.py
  - pochoir/units.py
  - README.org
updated: 2026-05-07
---

# source-pochoir-source

Deep source code investigation of `/nfs/data/1/xning/field_response/pochoir/`. All major Python source files read (excluding test scripts and generated data). Investigation context: understanding how PDVD field response files are produced from first principles.

## Pages created

- [[Pochoir FDM Solver]] — Jacobi stencil, 5 engine backends, cumba CUDA kernel (8×8×16 blocks), convergence criterion, edge conditions
- [[Pochoir LAr Physics]] — BNL mobility polynomial (6 coefficients a0..a5), DL/DT Einstein relations, velocity field computation pipeline
- [[Pochoir Drift Integration]] — scipy.solve_ivp Radau (rtol=atol=1e-10), RGI trilinear velocity interpolation, solve_sde Euler-Maruyama, induce-30deg ±1.45 mm alternation, convertfr 1325-sample truncation + sign flip

## Pages updated

- [[PDVD Field Response Computation]] — Added source-level implementation notes section

## Confirmed findings

- FDM uses Jacobi relaxation: `φ = (1/2N) × Σ neighbors`, converges at `max_change < precision` — see [[Pochoir FDM Solver]]
- cumba engine: CUDA kernel with 8×8×16 thread blocks, convergence sampled every 100 iterations (vs every 1 in numpy) — see [[Pochoir FDM Solver]]
- LAr mobility: BNL rational polynomial with 6 coefficients; longitudinal diffusion has an extra `E × dµ/dE` Einstein correction term — see [[Pochoir LAr Physics]]
- `bc_interp.py`: hardcoded `z_idx=1100` for far-field Z-end boundary — see [[PDVD Field Response Computation]]
- `extendwf`: hardcoded `cut_z=1100`; `onestrip = domain3d.shape[0] / 7`; 3 regions (i<7×os, 7×os≤i<14×os, i≥14×os) — see [[PDVD Field Response Computation]]
- `induce-30deg`: strip spacing `dx=7.615 mm`; ±1.45 mm Y-offset alternation for even/odd strips; 60 paths split 0..29 / 30..59 — see [[Pochoir Drift Integration]]
- `convertfr`: truncates to 1325 samples; stored current = `−1 × computed` (Ramo sign convention) — see [[Pochoir Drift Integration]]
- WireCell FR schema: `FieldResponse(planes, axis, origin, tstart, period, speed)`, `PlaneResponse(paths, planeid, location, pitch)`, `PathResponse(current, pitchpos, wirepos)` — see [[Pochoir Drift Integration]]
- Units base: `mm=1`, `ns=1`, `V=1e-6 MeV/eplus` — see [[Pochoir LAr Physics]]

## Context

PDVD field response computation pipeline was previously documented at a high level from the shell script. This session adds source-level implementation details: actual algorithms, constants, formulas, and data layout.
