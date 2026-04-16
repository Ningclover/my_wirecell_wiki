---
tags: [source]
type: file
updated: 2026-04-14
---

# Source: ProtoDUNE-VD WireCell Configuration

Jsonnet configuration files that control the actual WireCell processing pipeline for ProtoDUNE-VD (ProtoDUNE Vertical Drift).

**Location:** `raw/dunereco/dunereco/DUNEWireCell/protodunevd/`

## Files

| File | Role |
|------|------|
| `params.jsonnet` | Detector geometry, ADC, electronics parameters (data) |
| `simparams.jsonnet` | Simulation overrides (drift speed, diffusion, lifetime) |
| `nf.jsonnet` | Noise filtering component assembly |
| `sp.jsonnet` | Signal processing component assembly |
| `sp-filters.jsonnet` | Filter kernel parameters for SP |
| `chndb-base.jsonnet` | Channel noise database configuration |
| `chndb-perfect.jsonnet` | Perfect (MC) channel DB (no bad channels) |
| `chndb-resp.jsonnet` | Response-based channel DB |
| `img.jsonnet` | Wire-cell imaging pipeline (slicing, tiling, solving) |
| `dnnroi.jsonnet` | DNN-based ROI finding |
| `funcs.jsonnet` | Utility functions (channel maps, drift velocity, fanpipe) |
| `magnify-sinks.jsonnet` | Diagnostic ROOT file sinks |
| `wcls-nf-sp.jsonnet` | **Entry point**: NF + SP (with optional DNN ROI) |
| `wcls-nf.jsonnet` | **Entry point**: NF only |
| `wcls-nf-sp-img.jsonnet` | **Entry point**: NF + SP + Imaging |
| `wcls-rawdigit-sp.jsonnet` | SP from raw digits |
| `wcls-sim-drift-*.jsonnet` | Simulation entry points |
| `wct-sim-check*.jsonnet` | Simulation check pipelines |

## Data files

| File | Contents |
|------|----------|
| `protodunevd-wires-larsoft-v3.json.bz2` | Wire geometry |
| `PDVD_strip_length.json.bz2` | Per-strip lengths for shield coupling subtraction |
| `protodunevd_FR_norminal_260324.json.bz2` | Field response (updated 2026-03-24 by xning) |
| `pdvd-bottom-noise-spectra-v1.json.bz2` | Bottom drift noise spectra |
| `pdvd-top-noise-spectra-v2.json.bz2` | Top drift noise spectra (v2) |
| `microboone-charge-error.json.bz2` | Charge error model (from MicroBooNE, used in imaging) |

## See also

[[ProtoDUNE-VD WireCell Configuration Overview]], [[PDVD Detector Parameters]], [[PDVD Noise Filtering Configuration]], [[PDVD Signal Processing Configuration]], [[PDVD SP Filters]], [[PDVD Imaging Configuration]], [[PDVD DNN ROI]]
