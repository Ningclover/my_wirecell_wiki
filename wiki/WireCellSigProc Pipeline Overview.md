---
tags: [algorithm, component]
sources: 1
updated: 2026-04-14
---

# WireCellSigProc Pipeline Overview

The `sigproc` module converts raw ADC waveforms from LArTPC sense wires into calibrated charge signals for 3D reconstruction.

Reference: arXiv:1802.08709 (MicroBooNE collaboration).

## Two-stage pipeline

```
Raw ADC Frame
    │
    ▼
[Stage 1: Noise Filtering]   OmnibusNoiseFilter / OmnibusPMTNoiseFilter
    │
    ▼
Clean ADC Frame
    │
    ▼
[Stage 2: Signal Processing] OmnibusSigProc
    │
    ▼
Deconvolved Charge Frame (with ROI masks)
```

## Stage 1: Noise Filtering

Three-pass architecture in [[Omnibus Noise Filter]]:

1. **Per-channel filters** — chirp detection, sticky-code repair, RC correction, spectral masking, baseline subtraction
2. **Grouped filters** — coherent noise subtraction (compute median across ASIC group, subtract with signal protection)
3. **Per-channel status** — final noisy channel tagging

Optional: [[PMT Noise Filter]] removes PMT cross-talk into TPC wires.

## Stage 2: Signal Processing

Per wire plane (U induction, V induction, W collection), [[OmnibusSigProc]] runs:

1. **Response initialization** — field response ⊗ electronics response, redigitized to frame tick period
2. **Load + rebase** — traces into padded Eigen 2D matrix, optional linear baseline subtraction
3. **2D deconvolution** — FFT(time) → per-channel response correction → FFT(wire) → divide 2D field response → wire filter → IFFT(wire) → IFFT(time)
4. **[[ROI Formation]]** — tight ROIs (high threshold) + loose ROIs (low threshold, rebinned)
5. **[[ROI Refinement]]** — multi-stage iterative cleanup (BFS, multi-plane consistency, BreakROI, ShrinkROI, ExtendROI)
6. **Final filtering** — Wiener + Gaussian filter within ROIs → output charge traces

## Optional: L1SP Filter

For **shorted wire regions** (induction + collection electrically connected), standard deconvolution fails. [[L1SP Filter]] uses L1-regularized sparse deconvolution (LASSO) to jointly decompose the mixed signal.

## Key data structures

| Structure | Role |
|-----------|------|
| `IFrame` / `SimpleFrame` | Container of waveform traces with metadata |
| `ITrace` / `SimpleTrace` | Single channel: channel ID + tick offset + samples |
| `Eigen::Array2D` (`m_r_data`) | 2D matrix [wire × tick] for per-plane processing |
| `SignalROI` | Region of interest: channel, time range, charge |
| `IChannelNoiseDatabase` | Per-channel noise parameters and filter spectra |

## Configuration

Configured via Jsonnet. Key files:
- `adc-noise-sig.jsonnet` — full pipeline test
- `check_pdsp_sim_sp.jsonnet` — ProtoDUNE-SP sim

## Detector implementations

| Detector | Unique noise features |
|----------|-----------------------|
| MicroBooNE | Chirp, ADC bit shift, partial RC, coherent noise |
| ProtoDUNE-SP | Sticky codes, ledge artifacts, 50 kHz harmonic, FEMB clock |
| ProtoDUNE-HD | FEMB negative pulses, wide adaptive baseline (512 ticks) |
| ProtoDUNE-VD | Shield coupling subtraction (see [[ProtoDUNE-VD WireCell Configuration Overview]]) |
| DUNE CRP | RC undershoot, partial RC |
| ICARUS | RC undershoot only (simplest) |

## Sources

- [[source-sigproc-examination]]
