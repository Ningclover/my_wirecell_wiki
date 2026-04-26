---
tags: [algorithm]
sources: 2
updated: 2026-04-25
---

# ROI Refinement

Multi-stage iterative cleanup of ROIs identified by [[ROI Formation]]. The largest file in `sigproc` (~3,155 lines).

**File:** `src/ROI_refinement.cxx`  
**Header:** `src/ROI_refinement.h`  
**Class:** `WireCell::SigProc::ROI_refinement`

## Purpose

Remove fake signal ROIs while preserving real ones. Uses multi-plane geometric consistency and peak-finding to produce clean, well-bounded regions for final filtering.

## Construction

```cpp
ROI_refinement roi_refine(
    m_wanmm,
    nwire_u, nwire_v, nwire_w,
    m_r_th_factor,                    // r_th_factor = 3.0 (2.5 for APA0 in upstream PDHD config)
    m_r_fake_signal_low_th,           // = 500 e⁻
    m_r_fake_signal_high_th,          // = 1000 e⁻
    m_r_fake_signal_low_th_ind_factor,  // = 1.0 (local PDHD: 0.15 for APA0)
    m_r_fake_signal_high_th_ind_factor, // = 1.0 (local PDHD: 0.15 for APA0)
    m_r_pad, m_r_break_roi_loop,
    m_r_th_peak, m_r_sep_peak,
    m_r_low_peak_sep_threshold_pre,
    m_r_max_npeaks, m_r_sigma, m_r_th_percent, m_isWrapped
);
```

## Relationship maps maintained

- `front_rois[roi]` — ROIs on the next wire that overlap in time
- `back_rois[roi]` — ROIs on the previous wire that overlap in time
- `contained_rois[loose_roi]` — tight ROIs contained within a loose ROI

## Key parameters and their roles

### `r_th_factor` (default 3.0; upstream uses 2.5 for PDHD APA0)

Used in **two places**:

1. **`load_data()` — tight ROI admission gate** (`ROI_refinement.cxx:321`):
   ```cpp
   float threshold = plane_rms.at(irow) * th_factor;
   if (tight_roi->get_above_threshold(threshold).size() == 0) {
       delete tight_roi;   // discard: no tick exceeds th_factor × RMS
       continue;
   }
   ```
   Every tight ROI from `ROI_formation` is tested. If no single tick exceeds `r_th_factor × per-wire-RMS`, the ROI is discarded before entering refinement.

2. **`CheckROIs()` — inter-ROI connectivity pruning** (`ROI_refinement.cxx:1191`):
   ```cpp
   th  = th_factor * rms_u.at(chid);
   th1 = th_factor * rms_u.at(chid1);
   if (!roi->overlap(roi1, th, th1)) { unlink(roi, roi1); }
   ```
   ROIs on adjacent wires stay linked only if their overlap region reaches `r_th_factor × RMS` on both wires. Lower value → more weak induction signals survive connectivity pruning.

### `r_fake_signal_low_th` and `r_fake_signal_high_th`

Absolute electron thresholds for fake-signal cleanup. Applied differently for collection vs. induction:

**Collection (W) — `CleanUpCollectionROIs()`** (`ROI_refinement.cxx:1291`):
```
mean_threshold = fake_signal_low_th    (500 e⁻)
threshold      = fake_signal_high_th   (1000 e⁻)
// ind_factor NOT applied
```

**Induction (U/V) — `CleanUpInductionROIs()`** (`ROI_refinement.cxx:1373`):
```
mean_threshold = fake_signal_low_th  × fake_signal_low_th_ind_factor
threshold      = fake_signal_high_th × fake_signal_high_th_ind_factor
```

A ROI is declared **Good** if either:
- At least one tick exceeds `threshold` (peak height gate), OR
- `get_average_heights() > mean_threshold` (mean charge gate)

**Bad** ROIs (neither condition) are removed unless connected to a Good ROI via front/back links.

### Effect of `_ind_factor = 0.15` (PDHD APA0 local config anomaly)

With `ind_factor = 0.15`:
- mean threshold: 500 × 0.15 = **75 e⁻** (vs 500 upstream)
- peak threshold: 1000 × 0.15 = **150 e⁻** (vs 1000 upstream)

At 75/150 e⁻, nearly every noise fluctuation qualifies as Good → almost no induction ROIs are cleaned up. This factor (1/0.15 ≈ 6.7) matches the "6× too low" `summary_wiener` observed on APA0 V-plane.

## Pipeline stages

### 0. `load_data(plane, r_data, roi_form)`

Creates `SignalROI` objects from tight ROIs (with admission gate via `r_th_factor × RMS`). Builds loose ROI `SignalROI` objects from `roi_form.get_loose_rois()`. Establishes `front_rois`/`back_rois` connectivity graph between adjacent-wire overlapping ROIs.

### 1. `CleanUpROIs(plane)`

BFS flood-fill from tight ROIs. Any loose ROI not reachable (directly or transitively) from a tight ROI is removed.

### 2. `generate_merge_ROIs(plane)`

If a tight ROI is not contained in any loose ROI, promote it (create a new loose ROI to cover it).

### 3. `MP3ROI` / `MP2ROI` (optional, `use_multi_plane_protection`)

Multi-plane geometric protection: signal regions must be consistent across 3 (or 2) wire planes. Removes ROIs that don't have corresponding signal in other planes.

### 4. `BreakROIs(plane, roi_form)` × `r_break_roi_loop` iterations (default 2)

TSpectrum-like peak finding (via `PeakFinding`) to split wide ROIs at valleys. Each iteration followed by `CheckROIs` + `CleanUpROIs`.

### 5. `ShrinkROIs(plane, roi_form)`

Tighten ROI boundaries: trim edges where neighbor ROIs don't support the current extent.

### 6. `CleanUpCollectionROIs()` / `CleanUpInductionROIs(plane)`

Final S/N gate using `fake_signal_{low,high}_th × ind_factor`. See parameter section above.

### 7. `ExtendROIs(plane)`

Extend each surviving ROI to cover time range of overlapping neighbor ROIs. Ensures consistent coverage for baseline subtraction.

### 8. Final: `decon_2D_hits` + `apply_roi` (called from OmnibusSigProc)

After `ExtendROIs`, `OmnibusSigProc` applies `Wiener_wide` filter (`decon_2D_hits`) then zeros `m_r_data` outside ROIs (`roi_refine.apply_roi()`). Same mask applied to Gauss filter output.

## Known issues / bugs

- **BUG-CORE-1** (FIXED): Division by zero in `apply_roi` when `start_bin == end_bin`.
- **BUG-CORE-2** (FIXED): Division by zero in `BreakROI` when two consecutive valley positions coincide.
- **BUG-CORE-4** (NOT FIXED): Float-to-int truncation in baseline subtraction (up to 1 ADC count).
- **BUG-CORE-6** (FIXED): Unsigned underflow in `section_protected` — `saved_b.size()-1` wraps to UINT_MAX when empty.
- **BUG-CORE-7** (NOT FIXED): Fixed-size stack array `float valley_pos[205]` in `BreakROI`. If `max_npeaks > 200`, stack overflow.
- **BUG-CORE-8** (FIXED): Float comparison for integer wrap boundary in `nwire_w / 2.`.
- **BUG-CORE-10** (NOT FIXED, dead code): `TestROIs` boundary access would OOB, but not called in production.
- **EFF-CORE-2** (NOT FIXED): 3× code duplication for U/V/W planes.
- **EFF-CORE-3** (FIXED): Cached `bad_ch_map` lookups in inner loops.
- **EFF-CORE-5** (FIXED): `get_mp2_rois()` non-const overload returns by reference instead of copying map.
- **EFF-CORE-7** (NOT FIXED): `SignalROI` managed with raw `new`/`delete` — leak risk on exception paths.

## See also

- [[OmnibusSigProc]]
- [[ROI Formation]]
- [[PDHD Signal Processing Configuration]]

## Sources

- [[source-sigproc-examination]]
- [[source-session-2026-04-25-pdhd-sp-deconvolution]]
