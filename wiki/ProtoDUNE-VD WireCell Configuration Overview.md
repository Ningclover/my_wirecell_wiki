---
tags: [concept, experiment]
sources: 1
updated: 2026-04-14
---

# ProtoDUNE-VD WireCell Configuration Overview

Jsonnet-based configuration that instantiates the full WireCell processing pipeline for ProtoDUNE-VD. The configuration is split into modular files that are composed at runtime.

## Detector layout

- **8 anodes** (indices 0–7), each anode is one CRU (3072 channels)
- **Bottom drift**: anodes 0–3
- **Top drift**: anodes 4–7
- Each CRP (Charge Readout Plane) spans 2 anodes (CRU0 + CRU1)

## Entry points

| File | Pipeline | Key extVars |
|------|----------|-------------|
| `wcls-nf-sp.jsonnet` | NF → SP (optional DNN ROI) | `reality`, `epoch`, `signal_output_form`, `use_dnnroi`, `dnn_roi_model`, `inference_service`, `triton_url`, `use_resampler`, `use_magnify`, `use_ssl` |
| `wcls-nf.jsonnet` | NF only | `reality`, `signal_output_form`, `raw_input_label`, `use_resampler`, `use_magnify` |
| `wcls-nf-sp-img.jsonnet` | NF → SP → Imaging | `reality`, `signal_output_form`, `raw_input_label`, `use_magnify` |

`reality` = `"data"` loads `params.jsonnet`; `reality` = `"sim"` loads `simparams.jsonnet`.

## Full pipeline (wcls-nf-sp.jsonnet)

```
RawDigits (art::Event)
    │
    ▼
wclsRawFrameSource           [tag: orig]
    │
    ▼
FrameFanout (8-way)          one branch per anode
    │
    ├─ ChannelSelector        select channels for anode N
    ├─ [Resampler]            512ns→500ns, bottom only (n<4), if use_resampler
    ├─ [magnify orig]         optional diagnostic sink
    ├─ OmnibusNoiseFilter     NF: PDVDOneChannelNoise + PDVDCoherentNoiseSub
    │                              + PDVDShieldCouplingSub (top only)
    ├─ [magnify raw]
    ├─ OmnibusSigProc         SP: deconvolution + ROI pipeline
    ├─ [magnify decon]
    └─ [DNNROIFinding]        optional, U+V planes; W shunted via PlaneSelector
    │
    ▼
FrameFanin (8-way)
    │
    ▼
Retagger                     merge gaussN→gauss, wienerN→wiener
    │
    ▼
wclsFrameSaver               save recob::Wire [gauss, wiener]
```

## Frame tags

| Tag | Contents |
|-----|----------|
| `origN` | Raw ADC for anode N |
| `rawN` | Noise-filtered ADC for anode N |
| `gaussN` | Deconvolved charge (Gaussian filter) — best for charge reco |
| `wienerN` | Deconvolved charge (Wiener filter) — best for S/N |
| `thresholdN` | SP ROI threshold map |
| `decon_chargeN` | Intermediate deconvolved charge |
| `loose_lfN` | Loose low-frequency ROI (used by DNN ROI) |
| `mp2_roiN`, `mp3_roiN` | Multi-plane ROI maps (used by DNN ROI) |
| `dnnspNU/V/W` | DNN ROI output per plane |
| `gauss`, `wiener` | Final merged tags (post-retagger) saved to LArSoft |

## Output scaling

SP signals saved with `frame_scale = 0.005` (both gauss and wiener).

## Key design notes

- `MegaAnodePlane` used for output (spans all 8 anodes) rather than per-anode saves
- `chndb-base.jsonnet` used for real data (not `chndb-perfect.jsonnet` which is for ideal MC)
- Resampler only applied to bottom drift (n<4); top drift data already at 500ns
- DNN ROI mode disables multi-plane protection in SP (`use_multi_plane_protection: false`) — the DNN takes over ROI selection

## See also

[[PDVD Detector Parameters]], [[PDVD Noise Filtering Configuration]], [[PDVD Signal Processing Configuration]], [[PDVD Imaging Configuration]], [[PDVD DNN ROI]], [[Detector-Specific Signal Processing]]

## Sources

- [[source-pdvd-wct-config]]
