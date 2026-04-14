---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD Signal Processing Configuration

OmnibusSigProc configuration for ProtoDUNE-VD, defined in `sp.jsonnet`. One instance per anode.

## Electronics response selection

Bottom (anode.ident < 4) and top (anode.ident ≥ 4) use different electronics responses:

| Drift | Electronics response |
|-------|---------------------|
| Bottom (0–3) | `tools.elec_resps[0]` — standard shaping (7.8 mV/fC, 2.2 µs) |
| Top (4–7) | `tools.elec_resps[1]` — custom JSON: `dunevd-coldbox-elecresp-top-psnorm_400.json.bz2` |

## ADC conversion

```
ADC_mV = (2^14 - 1) / fullscale
```
- Bottom: `fullscale = 1.4 V` → ADC_mV ≈ 11,674 ADC/V
- Top: `fullscale = 2.0 V` → ADC_mV ≈ 8,191 ADC/V

## Timing offsets

| Parameter | Value | Note |
|-----------|-------|------|
| `ftoffset` | 0.0 | Field-to-time offset |
| `ctoffset` | 4 µs | Collection time offset — **must match** field response `protodunevd_FR_norminal_260324.json.bz2` |

## ROI thresholds

| Parameter | Value | Default | Meaning |
|-----------|-------|---------|---------|
| `troi_col_th_factor` | 5.0 | 5 | Tight ROI threshold factor, collection plane |
| `troi_ind_th_factor` | 3.0 | 3 | Tight ROI threshold factor, induction planes |
| `lroi_rebin` | 6 | 6 | Rebin factor for loose ROI |
| `lroi_th_factor` | 3.5 | 3.5 | Loose ROI threshold factor |
| `lroi_th_factor1` | 0.7 | 0.7 | Loose ROI secondary threshold |
| `lroi_jump_one_bin` | 1 | 0 | Allow jumping one bin in loose ROI (PDVD non-default) |
| `r_th_factor` | 3.0 | 3 | ROI refinement threshold |

## Fake signal rejection thresholds

| Parameter | Value | Default | Meaning |
|-----------|-------|---------|---------|
| `r_fake_signal_low_th` | 375 | 500 | Low fake signal threshold (reduced for PDVD) |
| `r_fake_signal_high_th` | 750 | 1000 | High fake signal threshold (reduced for PDVD) |
| `r_fake_signal_low_th_ind_factor` | 1.0 | 1 | Induction plane scale factor |
| `r_fake_signal_high_th_ind_factor` | 1.0 | 1 | Induction plane scale factor |
| `r_th_peak` | 3.0 | 3.0 | Peak detection threshold |
| `r_sep_peak` | 6.0 | 6.0 | Peak separation |
| `r_low_peak_sep_threshold_pre` | 1200 | 1200 | Low-peak separation pre-threshold |

## Other parameters

| Parameter | Value | Note |
|-----------|-------|------|
| `postgain` | 1.0 | (default is 1.2; PDVD uses 1.0) |
| `fft_flag` | 0 | Lower memory mode |
| `isWrapped` | false | Wire directions do not wrap |
| `use_multi_plane_protection` | true | Enabled by default |

## Frame tags (per anode N = anode.ident)

| Tag | Contents |
|-----|----------|
| `wiener{N}` | Wiener-filtered charge |
| `wiener_threshold{N}` / `threshold{N}` | Threshold map |
| `decon_charge{N}` | Intermediate deconvolved charge |
| `gauss{N}` | Gaussian-filtered charge |
| `tight_lf{N}` | Tight low-frequency ROI |
| `loose_lf{N}` | Loose low-frequency ROI |
| `cleanup_roi{N}` | Post-cleanup ROI |
| `break_roi_1st{N}` / `break_roi_2nd{N}` | BreakROI intermediate |
| `shrink_roi{N}` | Post-ShrinkROI |
| `extend_roi{N}` | Post-ExtendROI |
| `mp3_roi{N}` | 3-plane protection ROI |
| `mp2_roi{N}` | 2-plane protection ROI |

Debug tags (`tight_lf`, `cleanup_roi`, etc.) only written when `use_roi_debug_mode: true`.

## DNN ROI mode overrides

When `use_dnnroi=true`, `sp_override` from `wcls-nf-sp.jsonnet` changes:

| Parameter | DNN mode value | Normal value |
|-----------|---------------|--------------|
| `sparse` | false | false |
| `use_roi_debug_mode` | true | false |
| `use_multi_plane_protection` | true | true |
| `mp_tick_resolution` | 10 | 10 |
| `tight_lf_tag` | `""` (disabled) | `tight_lf{N}` |
| `cleanup_roi_tag` | `""` | `cleanup_roi{N}` |
| `break_roi_loop1_tag` | `""` | `break_roi_1st{N}` |
| `break_roi_loop2_tag` | `""` | `break_roi_2nd{N}` |
| `shrink_roi_tag` | `""` | `shrink_roi{N}` |
| `extend_roi_tag` | `""` | `extend_roi{N}` |
| `m_save_negative_charge` | false | false |

## See also

[[ProtoDUNE-VD WireCell Configuration Overview]], [[OmnibusSigProc]], [[PDVD SP Filters]], [[PDVD Detector Parameters]], [[PDVD DNN ROI]]

## Sources

- [[source-pdvd-wct-config]]
