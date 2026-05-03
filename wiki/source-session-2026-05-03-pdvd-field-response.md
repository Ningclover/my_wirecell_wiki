---
tags: [source]
type: conversation
date: 2026-05-03
context: Full PDVD field response pipeline walkthrough using pochoir toolkit
files_touched:
  - /nfs/data/1/xning/field_response/pochoir/test/test-full-3d-drift-3view_all_xn.sh
  - /nfs/data/1/xning/field_response/pochoir/test/helpers.sh
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_quarter_config_30deg.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_2D_config_30deg_u.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_2D_config_30deg_v.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_2D_config.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_3D_config_30deg_u.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_3D_config_30deg_v.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_gen_pcb_3D_config.json
  - /nfs/data/1/xning/field_response/pochoir/test/example_convertfr_vd.json
  - /nfs/data/1/xning/field_response/pochoir/pochoir/__main__.py
  - /nfs/data/1/xning/field_response/pochoir/pochoir/main.py
updated: 2026-05-03
---

# source-session-2026-05-03-pdvd-field-response

Full walkthrough of the PDVD (ProtoDUNE-VD) 3-view PCB strip field response computation pipeline using the `pochoir` toolkit. The main script is `test/test-full-3d-drift-3view_all_xn.sh`; the output store is `test/store_new_multi/`. The final product is `current/FR_xn_new.json.bz2` — a WireCell-format field response file.

## Confirmed findings

- 10-stage pipeline: domain → gen → FDM(2D) → bc-interp → FDM(3D) → extendwf → velo → starts/drift → induce → convertfr — see [[PDVD Field Response Computation]]
- `want` guard in `helpers.sh` checks for `.npz` existence before running any step — pipeline is idempotent — see [[PDVD Field Response Computation]]
- FDM uses `cumba` engine (CUDA+Numba); 2D fields run 20×200k iterations; 3D full fields run 10×9k iterations — see [[PDVD Field Response Computation]]
- `bc-interp` stitches 2D solution into 3D boundary at xcoord = (Nactive × stripX)/2: 26.78 mm for U/V, 17.85 mm for W — see [[PDVD Field Response Computation]]
- `extendwf` uses cut_z=1100 grid points as the 3D/2D switchover along drift axis — see [[PDVD Field Response Computation]]
- `induce-30deg` alternates Y-offset ±1.45 mm between even/odd strip indices to handle two orientations of the 30° unit cell — see [[PDVD Field Response Computation]]
- `convertfr` truncates FR waveforms to 1325 samples (132.5 µs at 100 ns period) and flips sign — see [[PDVD Field Response Computation]]
- The output FR file drives `ctoffset = 4 µs` in the PDVD `sp.jsonnet` signal processing config — see [[PDVD Signal Processing Configuration]]
- Geometry: PCB holes radius 1.2 mm, PCB width 3.2 mm; QuarterDimX=2.55 mm for 30° strips; strip pitches U/V=7.65 mm, W=5.1 mm — see [[PDVD Field Response Computation]]
- Drift simulation: 25×15×2100 grid at 0.1 mm spacing; T=89 K; start points at Z=198 mm; paths integrated 0–300 µs in 0.1 µs steps — see [[PDVD Field Response Computation]]

## Context

User requested a full-pipeline walkthrough of the PDVD field response computation script at `/nfs/data/1/xning/field_response/pochoir/test/test-full-3d-drift-3view_all_xn.sh` and all related pochoir source files, with results stored in `store_new_multi/`. Goal: document the complete 10-stage procedure in the wiki.
