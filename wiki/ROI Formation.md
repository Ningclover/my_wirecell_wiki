---
tags: [algorithm]
sources: 2
updated: 2026-04-25
---

# ROI Formation

Identifies signal regions (Regions of Interest) in deconvolved waveforms using a two-tier threshold approach. Also computes the per-wire noise RMS that propagates as `summary_wiener` to all downstream stages.

**File:** `src/ROI_formation.cxx` (937 lines)  
**Header:** `src/ROI_formation.h`  
**Class:** `WireCell::SigProc::ROI_formation`

## Purpose

After 2D deconvolution, the waveforms contain signal + noise. ROI formation finds contiguous time regions likely to contain real signal. The per-wire RMS computed here is the single shared noise estimate reused by [[ROI Refinement]] and [[OmnibusSigProc]] (as `summary_wiener`).

## Construction

```cpp
ROI_formation roi_form(
    m_wanmm,              // ChannelMaskMap — bad/noisy tick ranges
    nwire_u, nwire_v, nwire_w,
    m_nticks,             // nbins: time ticks per wire
    m_th_factor_ind,      // troi_ind_th_factor = 3.0 (U/V tight ROI threshold ×RMS)
    m_th_factor_col,      // troi_col_th_factor = 5.0 (W tight ROI threshold ×RMS)
    m_pad,                // troi_pad = 5 ticks, padding around tight ROIs
    m_asy,                // troi_asy = 0.1, asymmetry tolerance for connect-info
    m_rebin,              // lroi_rebin = 6, rebin factor for loose ROI
    m_l_factor,           // lroi_th_factor = 3.5, loose ROI threshold ×RMS×rebin
    m_l_max_th,           // lroi_max_th = 10000, cap on loose threshold
    m_l_factor1,          // lroi_th_factor1 = 0.7, fraction for ROI edge search
    m_l_short_length,     // = 3, short-peak window for local peak detection
    m_l_jump_one_bin      // lroi_jump_one_bin = 1/0, allow 1-bin gap at edges
);
```

Constructor also builds `bad_ch_map` from `cmm["bad"]` for fast bad-tick lookup.

## Stage 1: Tight ROIs + RMS — `find_ROI_by_decon_itself()`

Called per plane. The two-argument version (induction planes with refinement) takes both:
- `r_data` = tight-filtered decon (Wiener_tight × ROI_tight_lf)
- `r_data_tight` = tighter-filtered decon (Wiener_tight × ROI_tighter_lf)

The one-argument version (collection, or when `r_data_tight` not available) calls `find_ROI_by_decon_itself(plane, r_data, r_data)`.

For each wire `irow`:

### Step 1a: Per-wire RMS estimation (`cal_RMS`)

```
signal[] = r_data[irow, :] with bad-tick ranges excluded

Pass 1 — robust sigma from percentiles:
  par[0] = percentile(signal, 0.16)   // 16th percentile
  par[1] = percentile(signal, 0.50)   // median
  par[2] = percentile(signal, 0.84)   // 84th percentile
  rms_prelim = sqrt( ((par[2]-par[1])^2 + (par[1]-par[0])^2) / 2 )

Pass 2 — include only |sample| < 5×rms_prelim:
  rms = sqrt( mean(x^2) for samples within 5σ )
```

Result stored in `uplane_rms[irow]`, `vplane_rms[irow]`, or `wplane_rms[irow]`.

**This is the single per-wire noise estimate used by all downstream stages.**

### Step 1b: Tight ROI seed finding

```
threshold = th_factor_ind × rms + 1   (U/V planes; default th_factor_ind = 3.0)
threshold = th_factor_col × rms + 1   (W plane;    default th_factor_col = 5.0)

scan signal1[] (tight-filtered) and signal2[] (tighter-filtered):
if signal1[j] > threshold OR signal2[j] > threshold:
    start ROI, extend while either remains above threshold
    → raw tight ROI: [start, end]
```

### Step 1c: `extend_ROI_self(plane)`

Pad each raw ROI by ±`pad` (5) ticks. Merge overlapping ROIs.

### Step 1d: `create_ROI_connect_info(plane)`

For each wire `i`, compare wires `i` and `i+2`. If they each have ROIs of similar length (length difference < `asy × (len1+len2)`), interpolate a ROI onto wire `i+1` (the midpoint in time). Fills gaps on inefficient induction wires.

## Stage 2: Loose ROIs — `find_ROI_loose()`

Induction planes only. Applied after `decon_2D_looseROI()` (wider LF passband).

1. **Rebin** by `rebin=6`: `signal2[i] = sum(signal1[6i..6i+5])`
2. **Threshold**: `th = cal_RMS(signal) × rebin × l_factor` (capped at `l_max_th=10000`)
   - Note: this locally re-computes RMS on the loose-filtered waveform for peak detection only. The result is **not stored** as `[u/v/w]plane_rms` — only the Stage 1 RMS is stored.
3. **Peak detection**: scan rebinned signal; trigger on bins exceeding `th`. Also detect peaks below threshold using local max + prominence check.
4. **ROI edges**: walk outward using `find_ROI_begin/end()` until local 3-bin average stops decreasing by 25 (with optional 1-bin jump if `l_jump_one_bin=1`).
5. **Merge**: adjacent loose ROIs merged if valley between them stays above `th × l_factor1 = th × 0.7`.
6. **Scale back** to original ticks (×6).
7. **`extend_ROI_loose()`**: snap loose ROI edges to encompass any overlapping tight ROI boundaries.

## `apply_roi(plane, r_data, flag)`

Zeros `r_data` outside ROIs (flag=1: tight, flag=2: loose). Within each ROI:
- U/V planes: subtract a linear baseline connecting the ROI boundary values
- W plane: keep contents as-is (no baseline subtraction)

## Outputs

| Accessor | Content |
|---|---|
| `get_uplane_rms()` / `get_vplane_rms()` / `get_wplane_rms()` | Per-wire 1σ RMS from Stage 1 (`cal_RMS`) |
| `get_self_rois(chid)` | Tight ROI list `[(start,end), ...]` for channel |
| `get_loose_rois(chid)` | Loose ROI list for channel |

## RMS propagation chain

```
ROI_formation::find_ROI_by_decon_itself()
  → cal_RMS() on tight-filtered decon
  → [u/v/w]plane_rms[wire] stored

OmnibusSigProc:
  perplane_thresholds[3] = {&uplane_rms, &vplane_rms, &wplane_rms}
  perwire_rmses = *perplane_thresholds[iplane]

  → roi_refine.load_data():     tight ROI gate = r_th_factor × rms
  → roi_refine.CheckROIs():     link gate      = r_th_factor × rms
  → roi_refine.ShrinkROIs():    shrink boundary = r_th_factor × rms
  → save_data("wiener"):        threshold[trace] = rms
       → summary_wiener<N>_<ident>.npy (one value per trace)
       → MaskSlices: cut = nthreshold[plane] × rms (= 3.6σ) for imaging
```

## Known issues / bugs

- **BUG-CORE-1** (FIXED): Division by zero in `apply_roi` when `start_bin == end_bin` (single-bin ROI).
- **BUG-CORE-3** (FIXED): Dead merge loop — `flag_repeat` initialized to 0 before `while(flag_repeat)`, so iterative loose ROI merge never ran. Fixed: initialize to 1.
- **BUG-CORE-4** (NOT FIXED): Float-to-int truncation in baseline subtraction (up to 1 ADC count systematic bias).
- **EFF-CORE-1** (FIXED): `cal_RMS` took waveform by value (~10K floats copied per call). Changed to `const&`.
- **EFF-CORE-2** (NOT FIXED): Logic copy-pasted 3× for U/V/W planes — maintenance hazard.
- **EFF-CORE-3** (FIXED): `bad_ch_map` looked up 3× per column in inner loops. Now cached.

## See also

- [[OmnibusSigProc]]
- [[ROI Refinement]]
- [[PDHD Signal Processing Configuration]]

## Sources

- [[source-sigproc-examination]]
- [[source-session-2026-04-25-pdhd-sp-deconvolution]]
