---
tags: [algorithm]
sources: 1
updated: 2026-04-14
---

# ROI Formation

Identifies signal regions (Regions of Interest) in deconvolved waveforms using a two-tier threshold approach.

**File:** `src/ROI_formation.cxx` (937 lines)  
**Header:** `src/ROI_formation.h`

## Purpose

After 2D deconvolution, the waveforms contain signal + noise. ROI formation finds contiguous regions likely to contain real signal, which are then refined by [[ROI Refinement]].

## Method

### Tight ROIs (high threshold)
- Threshold: `threshold = th_factor × RMS + 1`
- Contiguous above-threshold bins, padded by `pad` ticks, merged when overlapping
- Represent clear, high-confidence signal

### Loose ROIs (low threshold)
- Waveform rebinned by factor `rebin`
- Threshold: `l_factor × RMS × rebin`
- Extended to cover tight ROI boundaries
- Represent possible signal with broader coverage

### RMS estimation
Two-pass robust sigma-clipping:
1. IQR-based initial estimate
2. Standard deviation of samples within the IQR range

### Gap channel interpolation
`create_ROI_connect_info()` — if a channel has no ROIs but both neighbors do, interpolate ROIs across the gap.

## Outputs

- Per-wire `SignalROIList` (both tight and loose)
- `front_rois` / `back_rois` maps for wire-to-wire ROI adjacency
- `contained_rois` map: tight ROIs contained within each loose ROI

## Known issues / bugs

- **BUG-CORE-1** (FIXED): Division by zero in `apply_roi` when `start_bin == end_bin` (single-bin ROI). Guarded with ternary.
- **BUG-CORE-3** (FIXED): Dead merge loop — `flag_repeat` initialized to 0 before `while(flag_repeat)`, so iterative loose ROI merge never ran. Fixed: initialize to 1.
- **BUG-CORE-4** (NOT FIXED): Float-to-int truncation in baseline subtraction (up to 1 ADC count systematic bias).
- **EFF-CORE-1** (FIXED): `cal_RMS` took waveform by value (~10K floats copied per call, thousands of times per frame). Changed to `const&`.
- **EFF-CORE-2** (NOT FIXED): Logic copy-pasted 3× for U/V/W planes — maintenance hazard.
- **EFF-CORE-3** (FIXED): `bad_ch_map` looked up 3× per column in inner loops. Now cached.

## See also

[[OmnibusSigProc]], [[ROI Refinement]], [[SignalROI]]

## Sources

- [[source-sigproc-examination]]
