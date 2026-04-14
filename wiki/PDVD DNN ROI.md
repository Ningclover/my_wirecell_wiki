---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD DNN ROI

DNN-based Region of Interest finding for ProtoDUNE-VD, configured in `dnnroi.jsonnet`. An alternative to the classical multi-plane protection ROI selection in [[OmnibusSigProc]].

## When enabled

Activated via `use_dnnroi=true` in `wcls-nf-sp.jsonnet`. Adds a `DNNROIFinding` node after SP for each anode.

## Architecture

For each anode, a 3-way fanout feeds U, V, W branches:

```
OmnibusSigProc output
    │
    ▼
FrameFanout (3-way)
    ├── DNNROIFinding (plane=0, U)   → tag: dnnsp{N}u
    ├── DNNROIFinding (plane=1, V)   → tag: dnnsp{N}v
    └── PlaneSelector (plane=2, W)   → tag: dnnsp{N}w  (W shunted, uses gauss tag)
    │
    ▼
FrameFanin (3-way)                   → merged tag: dnnsp{N}
```

## DNN inputs

`DNNROIFinding` for U and V planes receives:
- `loose_lf{N}` — loose low-frequency ROI from SP
- `mp2_roi{N}` — 2-plane multi-plane protection ROI
- `mp3_roi{N}` — 3-plane multi-plane protection ROI
- `decon_charge{N}` — intermediate deconvolved charge

These tags require `use_roi_debug_mode: true` in SP (set automatically when `use_dnnroi`).

## W plane handling

The W (collection) plane is **not** passed through a DNN. Instead, `PlaneSelector` extracts the W plane from `gauss{N}` and re-tags it as `dnnsp{N}w`. The final DNN output for W is just the classical Gaussian-filtered SP result.

## Inference service

Supports two backends, selected via `inference_service` extVar:

| Service | Config |
|---------|--------|
| `TorchService` | Local CPU inference; `model` = model file path; `concurrency: 1` |
| `TritonService` | Remote gRPC; `url` = triton server URL; `use_ssl` flag |

Model file specified via `dnn_roi_model` extVar.

## Output scale

`output_scale: 1.0` — DNN output multiplied by this before saving.

## SP behavior changes when DNN ROI enabled

`wcls-nf-sp.jsonnet` passes `sp_override` to disable classical intermediate tags (so SP does full processing but suppresses intermediate debug saves):

- Multi-plane protection kept ON (provides `mp2_roi`, `mp3_roi` as DNN inputs)
- `tight_lf_tag`, `cleanup_roi_tag`, `break_roi_loop*_tag`, `shrink_roi_tag`, `extend_roi_tag` all set to `""` to suppress those debug outputs

## See also

[[ProtoDUNE-VD WireCell Configuration Overview]], [[PDVD Signal Processing Configuration]], [[ROI Refinement]]

## Sources

- [[source-pdvd-wct-config]]
