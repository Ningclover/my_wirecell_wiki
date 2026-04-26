---
tags: [experiment, algorithm]
sources: 1
updated: 2026-04-26
---

# PDHD Signal Processing Configuration

ProtoDUNE-HD WireCell signal processing configuration: SP filter parameters, per-APA tuning, and config differences between local/upstream/validation repos.

**Config dir:** `wire-cell-cfg/pgrapher/experiment/pdhd/`  
**Key files:** `sp.jsonnet`, `params.jsonnet`, `nf.jsonnet`, `chndb-base.jsonnet`, `sp-filters.jsonnet`

## Detector geometry (PDHD APA)

- 4 APAs (anode 0–3), each with 2560 channels: 800 U + 800 V + 960 W
- ADC: 14-bit
- `m_nticks` per event: derived from NF output frame tick span (e.g. 5859 ticks for run 027409 evt 2)
- `m_fft_flag = 0` (forced off, broken): no FFT-optimal padding → `m_fft_nticks = m_nticks`, no wire padding

## Output archives from FrameFileSink

Each `FrameFileSink` produces a `.tar.bz2` per APA containing:

| File | Shape | Content |
|---|---|---|
| `frame_gauss<N>_<ident>.npy` | `(2560, nticks)` float32 | Gauss-filtered deconvolved charge |
| `frame_wiener<N>_<ident>.npy` | `(2560, nticks)` float32 | Wiener-filtered deconvolved charge |
| `channels_gauss<N>_<ident>.npy` | `(2560,)` int32 | WCT channel IDs |
| `channels_wiener<N>_<ident>.npy` | `(2560,)` int32 | WCT channel IDs |
| `tickinfo_*_<ident>.npy` | `(3,)` float64 | `[frame.time, frame.tick, tbinmin]` |
| `summary_wiener<N>_<ident>.npy` | `(2560,)` float64 | Per-trace noise RMS (= `[u/v/w]plane_rms` from ROI_formation) |
| `chanmask_bad_<ident>.npy` | `(N_bad, 3)` int32 | Bad channel masks `[ch, start, end]` |

`summary_wiener` is the `trace_summary` attached to wiener traces by `OmnibusSigProc` via `tag_traces`. Gauss traces carry no summary. Raw NF-output archive (`*-raw-anode<N>.tar.bz2`) contains only `frame_raw`, `channels_raw`, `tickinfo_raw`.

## SP filter assignments (sp.jsonnet / sp-filters.jsonnet)

| Filter role | Config key | Applied to |
|---|---|---|
| Tight ROI (induction LF) | `ROI_tight_lf_filter` | U/V tight + loose ROI finding |
| Tighter ROI (induction LF) | `ROI_tighter_lf_filter` | U/V tighter ROI (one pass stronger) |
| Loose ROI (induction LF) | `ROI_loose_lf_filter` | U/V loose ROI finding |
| Wiener tight HF | `Wiener_tight_filters[0/1/2]` | All decon filter passes |
| Wiener wide HF | `Wiener_wide_filters[0/1/2]` | Final Wiener output |
| Gauss wide HF | `Gaus_wide_filter` | Final Gauss output |
| Wire HF | `Wire_filters[0/1/2]` | Applied once in decon_2D_init |

## Key SP tuning parameters (sp.jsonnet)

| Parameter | Local (APA0) | Upstream / WCP-validation | Effect |
|---|---|---|---|
| `r_fake_signal_low_th_ind_factor` | **0.15** | 1.0 | Induction mean-charge fake-signal gate multiplier |
| `r_fake_signal_high_th_ind_factor` | **0.15** | 1.0 | Induction peak-charge fake-signal gate multiplier |
| `r_th_factor` | 3.0 (all APAs) | 2.5 for APA0, 3.0 others | ROI admission + connectivity SNR floor |
| `r_low_peak_sep_threshold_pre` | 1200 | 1200 | Pre-separation threshold for BreakROI |

The `_ind_factor = 0.15` on APA0 (local config) lowers effective induction thresholds to 75/150 e⁻ (vs 500/1000 upstream), causing nearly all noise ROIs to survive `CleanUpInductionROIs`. This was identified as the root cause of anomalous `summary_wiener` values on APA0 V/W channels in run 027409.


## `summary_wiener` → imaging threshold chain

```
ROI_formation::cal_RMS() on tight-filtered decon
  → per-wire RMS stored in [u/v/w]plane_rms[]
  → summary_wiener<N>_<ident>.npy via FrameFileSink
  → MaskSlices (img module): nthreshold[plane] × summary[idx]
     nthreshold = [3.6, 3.6, 3.6]   (from img.jsonnet:116)
     effective cut = 3.6σ per wire per tick
```

## Signal units

All `frame_wiener` and `frame_gauss` arrays are in **electrons per tick**. `summary_wiener` values are **electrons** (1σ noise RMS). See [[ADC to Electrons Signal Chain]] for the full conversion.

## See also

- [[OmnibusSigProc]]
- [[ROI Formation]]
- [[ROI Refinement]]
- [[ADC to Electrons Signal Chain]]
- [[PDVD Signal Processing Configuration]]

## Sources

- [[source-session-2026-04-25-pdhd-sp-deconvolution]]
- [[source-session-2026-04-26-adc-to-electrons]]
