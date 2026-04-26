---
tags: [algorithm, concept]
sources: 1
updated: 2026-04-26
---

# ADC to Electrons Signal Chain

Traces the absolute signal amplitude from raw ADC counts through noise filtering and 2D deconvolution to the final output in electrons per tick.

**Relevant files:** `sigproc/src/OmnibusSigProc.cxx`, `wire-cell-cfg/pgrapher/experiment/pdhd/sp.jsonnet`, `wire-cell-cfg/pgrapher/experiment/pdhd/params.jsonnet`

## Stage 0: Raw ADC (NF input)

- Input: 14-bit ADC counts from the digitizer
- PDHD fullscale: `[0.2 V, 1.6 V]` → range = **1400 mV** ([params.jsonnet](../filt_resp/wire-cell-cfg/pgrapher/experiment/pdhd/params.jsonnet))
- `PDHDOneChannelNoise::apply()` performs baseline subtraction (DC removal + median baseline) and flags lf_noisy / noisy channels
- After NF: signal is still in **ADC counts** (baseline-subtracted, noise-filtered)

## Stage 1: Combined field × electronics response construction (`init_overall_response`)

`OmnibusSigProc::init_overall_response()` builds `overall_resp[plane]` — the combined field and electronics response used for deconvolution (`OmnibusSigProc.cxx:772`).

The electronics response waveform is scaled before FFT (`OmnibusSigProc.cxx:853`):

```cpp
Waveform::scale(ewave, m_inter_gain * m_ADC_mV * (-1));
elec = fwd_r2c(m_dft, ewave);
```

Where:
- `m_inter_gain` = `postgain` = **1.0** (sp.jsonnet)
- `m_ADC_mV` = `ADC_mV_ratio` = `(2^resolution − 1) / fullscale` = `(2^14 − 1) / 1400 mV` ≈ **11.70 ADC/mV** (sp.jsonnet)
- The minus sign accounts for signal polarity convention (ionization electrons → amplifier sign)

PDHD electronics parameters (params.jsonnet):
- FE amplifier gain: **14 mV/fC**
- Shaping time: **2.2 µs**

The `overall_resp[plane]` entries thus have units of **ADC counts per electron** (integrated signal per tick), because:

```
overall_resp = FR [e⁻·s/cm] × elec_resp [mV/fC] × (1 fC / 6242 e⁻) × (11.70 ADC/mV)
             → units: ADC·ticks / e⁻
```

(The field response is built by `Response::wire_region_average(fr)` and resampled to DAQ tick period.)

## Stage 2: 2D deconvolution output → electrons/tick (`decon_2D_init`)

In frequency domain (`OmnibusSigProc.cxx:1121`):

```cpp
m_c_data[plane] = m_c_data[plane] / c_resp;
```

This divides out the combined response:

```
m_c_data = FFT(ADC waveform) / FFT(overall_resp)
         = FFT(ADC) / [FFT(FR) × FFT(elec) × ADC_mV_ratio]
```

After iFFT:

```
m_r_data[plane] → units: e⁻ per tick
```

All subsequent filter passes (tight/tighter/loose for ROI finding, Wiener_wide, Gaus_wide) are dimensionless filters in frequency space — they attenuate but **do not rescale** the units.

## Stage 3: Final output and derived quantities

| Quantity | Units | Source |
|---|---|---|
| `frame_wiener<N>_<ident>.npy` | e⁻ per tick | Wiener_wide × apply_roi on decon output |
| `frame_gauss<N>_<ident>.npy` | e⁻ per tick | Gaus_wide × apply_roi on decon output |
| `summary_wiener<N>_<ident>.npy` (noise RMS) | e⁻ | `cal_RMS()` on tight-filtered decon waveform |
| ROI threshold `r_fake_signal_low_th` | e⁻ | Local PDHD: 375 e⁻; upstream: 500 e⁻ |
| ROI threshold `r_fake_signal_high_th` | e⁻ | Local PDHD: 750 e⁻; upstream: 1000 e⁻ |
| `MaskSlices` imaging cut | e⁻ per tick | `3.6 × summary_wiener[wire]` |

## Full signal chain summary

| Stage | Signal | Units |
|---|---|---|
| Raw digitizer output | 14-bit ADC samples | ADC counts |
| After NF (baseline subtraction, noise removal) | Baseline-subtracted waveform | ADC counts |
| `overall_resp` construction | `FR × elec × ADC_mV_ratio` | ADC·tick / e⁻ |
| `decon_2D_init`: data / overall_resp in freq domain | Deconvolved charge | **e⁻ per tick** |
| Wiener/Gauss filter passes | Filtered charge | e⁻ per tick |
| `frame_wiener`, `frame_gauss` output | Tagged traces | e⁻ per tick |
| `summary_wiener` (noise RMS) | `cal_RMS()` on tight-filtered decon | **e⁻** (1σ) |
| `MaskSlices` imaging cut | `nthreshold[plane] × summary` | e⁻ per tick |

## Key numerical values (PDHD)

- ADC/mV conversion: `(2^14 − 1) / 1400 mV ≈ 11.70 ADC/mV`
- FE gain: 14 mV/fC → 1 e⁻ = 1/(6242) fC × 14 mV/fC ≈ **2.24 µV/e⁻** before digitization
- `postgain = 1.0`: no additional inter-stage amplification applied
- The per-channel response correction (`per_chan_resp`, from FEMB calibration) normalizes each channel's actual gain and shaping to the nominal before the common deconvolution — applied at `OmnibusSigProc.cxx:1085–1101`

## Sign convention

The `(-1)` in `ewave` scaling (`OmnibusSigProc.cxx:853`) is needed because:
- The Garfield field response (FR) is defined for ionization electrons drifting toward the wire, producing a physically defined induced current
- The cold electronics amplifier and ADC baseline are set up so that signal appears as a positive ADC excursion on collection and bipolar on induction
- The overall sign is embedded in the response model so that the deconvolved output comes out **positive for collected charge** (conventional e⁻ count)

## See also

- [[OmnibusSigProc]]
- [[ROI Formation]]
- [[PDHD Signal Processing Configuration]]

## Sources

- [[source-session-2026-04-26-adc-to-electrons]]
