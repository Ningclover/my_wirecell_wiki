---
tags: [component, algorithm]
sources: 2
updated: 2026-04-25
---

# OmnibusSigProc

Master orchestrator for WireCell signal processing. Processes each wire plane (U, V, W) independently through the full deconvolution + ROI pipeline.

**File:** `src/OmnibusSigProc.cxx` (~1,960 lines)  
**Header:** `inc/WireCellSigProc/OmnibusSigProc.h`

## Key member data

- `m_r_data[3]` — Eigen 2D arrays `[wire × tick]`, one per plane, holding waveform data
- `m_c_data[3]` — complex FFT counterparts of `m_r_data`
- `overall_resp[3]` — per-plane per-wire combined (field × electronics) response waveforms, length `m_fft_nticks`
- `m_nwires[3]` — wire counts per plane (set from anode geometry in `configure()`)
- `m_nticks`, `m_fft_nticks` — tick counts derived per-event from the input frame (NOT from config)
- `m_fft_nwires[3]`, `m_pad_nwires[3]` — FFT wire dimensions (equal to `m_nwires` when `m_fft_flag=0`)
- `m_wire_shift[3]` — per-plane circular wire shift to undo 2D convolution offset
- `m_wanmm` — `ChannelMaskMap` (OSP channel-indexed bad/noisy masks, converted from WCT channel IDs each event)

## `m_nticks` origin and scope

`m_nticks` is **not set from config** (a config value is explicitly ignored with a warning at line 67). It is computed per-event in `init_overall_response()`:

```
m_nticks = max(tbin + trace.size()) − min(tbin)  over all traces in the frame
```

This is the exact tick-span of NF output data for the event. With `m_fft_flag=0` (always for PDHD — the flag is forced off):
- `m_fft_nticks = m_nticks` (no padding)
- `m_fft_nwires[i] = m_nwires[i]` (no wire padding)
- `m_pad_nwires[i] = 0`

Everything sized by `m_nticks`:
- `m_r_data[plane]` allocated `(m_fft_nwires, m_fft_nticks)` in `load_data()`
- `m_c_data[plane]` — same dimensions, complex
- `overall_resp[plane]` entries — each waveform has `m_fft_nticks` samples
- All filter waveforms: `filter_waveform(m_c_data[plane].cols())` = size `m_fft_nticks/2+1`
- Time shift applied circularly over `m_fft_nticks` columns
- Output traces in `save_data()`: `ITrace::ChargeSequence charge(m_nticks, 0)`
- `roi_form` constructor `nbins` argument = `m_nticks`

## `operator()` pipeline overview

```
init_overall_response(in)       — build overall_resp, derive m_nticks, m_fft_nticks
ROI_formation roi_form(...)     — constructed once per frame
ROI_refinement roi_refine(...)  — constructed once per frame

for each plane:
  load_data(in, plane)          — fill m_r_data, zero bad tick ranges
  [apply filter_responses_tn]   — optional per-wire multiplicative filter on overall_resp
  decon_2D_init(plane)          — full 2D deconvolution → m_r_data, m_c_data refreshed

  if induction (U, V) and use_roi_refinement:
    decon_2D_tighterROI(plane)  — Wiener_tight × ROI_tighter_lf → m_r_data (save as r_data_tight)
    decon_2D_tightROI(plane)    — Wiener_tight × ROI_tight_lf → m_r_data
    roi_form.find_ROI_by_decon_itself(plane, m_r_data, r_data_tight)  ← computes RMS here
    decon_2D_looseROI(plane)    — Wiener_tight × ROI_loose_lf → m_r_data
    roi_form.find_ROI_loose(plane, m_r_data)
    decon_2D_ROI_refine(plane)  — Wiener_tight × ROI_tight_lf, restore_baseline
  else (W collection):
    decon_2D_tightROI(plane)    — Wiener_tight → m_r_data (no LF for collection)
    roi_form.find_ROI_by_decon_itself(plane, m_r_data)    ← computes RMS here

  roi_refine.load_data(plane, m_r_data, roi_form)   — load SignalROI objects

(cross-plane ROI refinement loop after all 3 planes loaded)
for each plane:
  roi_refine.CleanUpROIs → generate_merge_ROIs → [MP3/MP2ROI] →
    BreakROIs×2 → CheckROIs → CleanUpROIs →
    ShrinkROIs → CheckROIs → CleanUpROIs →
    CleanUp{Collection,Induction}ROIs → ExtendROIs

  decon_2D_hits(plane)          — Wiener_wide filter → m_r_data
  roi_refine.apply_roi(plane, m_r_data)   — zero outside ROIs
  save_data(..., "wiener", thresholds)    — thresholds = perwire_rmses from roi_form

  decon_2D_charge(plane)        — Gaus_wide filter → m_r_data
  roi_refine.apply_roi(plane, m_r_data)   — same ROI mask
  save_data(..., "gauss")                 — no threshold stored
```

## `init_overall_response()` — combined field × electronics response

Runs once per event (can run a second time per plane if `filter_responses_tn` set). Steps:

1. Load Garfield field response, compute wire-region average → `fravg`
2. For each plane: build 2D array `arr[fine_wire, fine_tick]`
3. FFT along time → multiply by `elec[f] × Δt_fine` → iFFT (combines field + electronics in frequency domain)
4. Apply `m_fine_time_offset` as a circular shift
5. Resample from fine time bins to DAQ tick period `m_period` via linear interpolation → store in `overall_resp[iplane]`
6. `m_wire_shift[iplane] = (n_fine_wires − 1) / 2`
7. If `filter_responses_tn` set: multiply each `overall_resp[iplane][wire]` element-wise by per-wire filter

## `decon_2D_init()` — 2D Wiener deconvolution (detailed)

Starting from NF-output waveforms in `m_r_data[plane]`:

```
1. pad_data(plane)
   → for detectors with physically separate wire sub-planes (isWrapped), pad zeros between
     groups so the 2D FFT treats them as one continuous plane.
     PDHD standard APA: m_nwires_separate_planes empty → no-op.

2. FFT along time (row-by-row):
   m_c_data[plane] = fwd_r2c(m_r_data[plane], axis=1)
   shape: (nwires, fft_nticks/2+1) complex

3. Per-channel electronics correction (if m_per_chan_resp set):
   for each wire: ch_resp = per-channel measured electronics response (FEMB calibration)
   m_c_data[row, f] *= elec_nominal[f] / ch_elec[f]
   → corrects channel-to-channel gain/shaping variations

4. FFT along wire (col-by-col):
   m_c_data[plane] = fwd(m_c_data[plane], axis=0)
   → fully 2D complex: (fft_nwires, fft_nticks/2+1)

5. Build 2D FFT of overall_resp[plane]:
   r_resp[fine_wire, tick] → fwd_r2c (time) → fwd (wire) → c_resp

6. Divide: m_c_data[plane] /= c_resp
   → 2D Wiener deconvolution kernel; divides out combined field × electronics response

7. Apply wire filter (HfFilter named "Wire_ind" / "Wire_col"):
   m_c_data[row, f] *= wire_filter_wf[row]
   → suppresses high-frequency cross-wire noise
   NaN/Inf cells zeroed here.

8. iFFT along wire → m_c_data stays complex (full time dimension)
9. iFFT along time → m_r_data[plane] = real part, shape (nwires, nticks)

10. Wire shift (circular):
    shift rows by m_wire_shift[plane] to undo 2D convolution phase offset

11. Time shift (circular):
    time_shift = (m_coarse_time_offset + m_intrinsic_time_offset) / m_period
    shift columns to align signal to correct tick

12. unpad_data(plane)  — undo step 1, strip padding rows

13. Re-FFT along time only:
    m_c_data[plane] = fwd_r2c(m_r_data[plane], axis=1)
    → keeps m_c_data current for subsequent filter applications
```

After `decon_2D_init`, `m_r_data[plane]` = initial 2D-deconvolved waveform (electrons/tick/wire).
All subsequent `decon_2D_*` functions apply a new time-domain filter to the existing `m_c_data` and `inv_c2r` → trim to `(m_nwires, m_nticks)`.

## Filter variants and when each is applied

| Function | Time filter | LF filter | For |
|---|---|---|---|
| `decon_2D_tightROI` | `Wiener_tight` | `ROI_tight_lf` (U/V) or none (W) | Tight ROI seed finding |
| `decon_2D_tighterROI` | `Wiener_tight` | `ROI_tighter_lf` | Induction tighter-ROI seed |
| `decon_2D_looseROI` | `Wiener_tight` | `ROI_loose_lf` + neighbor check | Loose ROI finding (induction only) |
| `decon_2D_ROI_refine` | `Wiener_tight` | `ROI_tight_lf` | After loose ROI found; baseline restored |
| `decon_2D_hits` | `Wiener_wide` | — | Final Wiener output (S/N optimized) |
| `decon_2D_charge` | `Gaus_wide` | — | Final Gauss output (charge optimized) |

`restore_baseline()` is called after `decon_2D_tightROI`, `decon_2D_tighterROI`, `decon_2D_looseROI`, and `decon_2D_ROI_refine`. It median-subtracts each wire's non-zero samples to correct any residual baseline.

`decon_2D_looseROI` additionally checks `masked_neighbors("bad", ..., n_bad_nn)` and `masked_neighbors("lf_noisy", ..., n_lfn_nn)`. If the wire or its neighbors are bad/LF-noisy, it falls back from the loose LF filter to the tight LF filter for that wire.

## `save_data()` and the per-wire RMS / threshold

Per-trace threshold (the value stored in `summary_wiener`):
```cpp
const float thresh = perwire_rmses[och.wire];   // = roi_form.[u/v/w]plane_rms[wire]
threshold.push_back(thresh);
```
- Only attached to wiener traces via `sframe->tag_traces(m_wiener_tag, wiener_traces, thresholds)`
- Gauss traces: `sframe->tag_traces(m_gauss_tag, gauss_traces)` — no summary
- Saved to disk as `summary_wiener<N>_<ident>.npy` by `FrameFileSink`
- Used downstream by `MaskSlices` as `nthreshold[plane] × summary[idx]` (3.6σ) to decide which ticks form imaging slices

## Known issues / bugs

- **BUG-CORE-5** (FIXED): Config key typos — `"r_th_precent"` (should be `"r_th_percent"`), `"isWarped"` (should be `"isWrapped"`).
- **BUG-CORE-9** (NOT FIXED): Possibly inverted interpolation weights in `init_overall_response` (lines 924-925). Needs physics validation.
- **EFF-CORE-4** (NOT FIXED): `c_data_afterfilter` full complex array allocated ~8 times per plane. Workspace reuse would help.
- **EFF-CORE-6** (NOT FIXED): Filter objects re-fetched via `Factory::find` on every invocation.

## Key configuration parameters

| Config key | Member | Default | Meaning |
|---|---|---|---|
| `troi_ind_th_factor` | `m_th_factor_ind` | 3.0 | Tight ROI threshold multiplier for U/V (× per-wire RMS) |
| `troi_col_th_factor` | `m_th_factor_col` | 5.0 | Tight ROI threshold multiplier for W (× per-wire RMS) |
| `troi_pad` | `m_pad` | 5 | Ticks to pad around each tight ROI |
| `troi_asy` | `m_asy` | 0.1 | Asymmetry tolerance for ROI connect-info interpolation |
| `lroi_rebin` | `m_rebin` | 6 | Rebin factor for loose ROI finding |
| `lroi_th_factor` | `m_l_factor` | 3.5 | Loose ROI threshold multiplier (× RMS × rebin) |
| `lroi_th_factor1` | `m_l_factor1` | 0.7 | Fraction of loose threshold for ROI edge search |
| `lroi_jump_one_bin` | `m_l_jump_one_bin` | 0 | Allow 1-bin gap when walking ROI edges |
| `r_th_factor` | `m_r_th_factor` | 3.0 | ROI refinement admission + connectivity threshold (× per-wire RMS) |
| `r_fake_signal_low_th` | `m_r_fake_signal_low_th` | 500 e⁻ | Mean charge gate for fake-signal cleanup |
| `r_fake_signal_high_th` | `m_r_fake_signal_high_th` | 1000 e⁻ | Peak charge gate for fake-signal cleanup |
| `r_fake_signal_low_th_ind_factor` | `m_r_fake_signal_low_th_ind_factor` | 1.0 | Multiplier on low_th for induction planes only |
| `r_fake_signal_high_th_ind_factor` | `m_r_fake_signal_high_th_ind_factor` | 1.0 | Multiplier on high_th for induction planes only |
| `r_break_roi_loop` | `m_r_break_roi_loop` | 2 | Iterations of BreakROIs+CheckROIs+CleanUpROIs |
| `isWrapped` | `m_isWrapped` | false | Whether wire directions wrap (affects ROI_refinement boundary logic) |
| `sparse` | `m_sparse` | false | If true, output traces are sparsified (only non-zero runs) |
| `use_roi_refinement` | `m_use_roi_refinement` | true | Enable full ROI refinement pipeline |

## See also

- [[WireCellSigProc Pipeline Overview]]
- [[ROI Formation]]
- [[ROI Refinement]]
- [[PDHD Signal Processing Configuration]]

## Sources

- [[source-sigproc-examination]]
- [[source-session-2026-04-25-pdhd-sp-deconvolution]]
