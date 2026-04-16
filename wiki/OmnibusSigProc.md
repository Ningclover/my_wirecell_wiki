---
tags: [component, algorithm]
sources: 1
updated: 2026-04-14
---

# OmnibusSigProc

Master orchestrator for WireCell signal processing. Processes each wire plane (U, V, W) independently through the full deconvolution + ROI pipeline.

**File:** `src/OmnibusSigProc.cxx` (1,925 lines)  
**Header:** `inc/WireCellSigProc/OmnibusSigProc.h`

## Key member data

- `m_r_data[3]` — Eigen 2D arrays [wire × tick], one per plane, holding waveform data
- `m_c_data[3]` — complex FFT counterparts
- `m_wiener_*`, `m_gauss_*` — filter names resolved via WCT Factory

## operator() pipeline per plane

1. `init_overall_response(in)` — convolve field response with electronics, redigitize to frame tick period
2. `load_data(traces, plane)` — populate `m_r_data`, zero bad channels
3. `decon_2D_init(plane)` — 2D FFT deconvolution (time + wire)
4. `decon_2D_tightROI(plane)` / `decon_2D_looseROI(plane)` — filtered versions for ROI threshold passes
5. `ROI_formation` — identify signal regions (tight + loose)
6. `ROI_refinement` — multi-stage iterative cleanup
7. Final Wiener / Gaussian filtering within ROIs → output traces

## 2D deconvolution (`decon_2D_init`)

- FFT in time (per wire) → apply per-channel response correction
- FFT in wire direction → divide by 2D field response
- Apply wire-direction filter → IFFT(wire) → IFFT(time)
- Time and wire shift corrections applied

## Known issues / bugs

- **BUG-CORE-5**: Config key typos — `"r_th_precent"` (should be `"r_th_percent"`), `"isWarped"` (should be `"isWrapped"`). **FIXED.**
- **BUG-CORE-9**: Possibly inverted interpolation weights in `init_overall_response` (lines 924-925). Needs physics validation. **NOT FIXED.**
- **EFF-CORE-4**: `c_data_afterfilter` full complex array allocated ~8 times per plane. Workspace reuse would help. **NOT FIXED.**
- **EFF-CORE-6**: Filter objects re-fetched via `Factory::find` on every invocation. **NOT FIXED.**

## Key configuration parameters

| Parameter | Meaning |
|-----------|---------|
| `r_th_percent` | ROI threshold fraction |
| `isWrapped` | Whether wire directions wrap around |
| `m_break_roi_loop` | Iterations for BreakROI (default 2) |
| `m_wiener_*` / `m_gauss_*` | Named filter waveform objects |

## See also

- [[WireCellSigProc Pipeline Overview]]
- [[ROI Formation]]
- [[ROI Refinement]]

## Sources

- [[source-sigproc-examination]]
