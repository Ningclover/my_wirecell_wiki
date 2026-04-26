---
tags: [source]
type: conversation
date: 2026-04-26
context: Tracing the absolute signal amplitude through the full PDHD WireCell pipeline — raw ADC → noise filtering → 2D deconvolution → electrons/tick output
files_touched:
  - sigproc/src/OmnibusSigProc.cxx
  - sigproc/src/ProtoduneHD.cxx
  - wire-cell-cfg/pgrapher/experiment/pdhd/sp.jsonnet
  - wire-cell-cfg/pgrapher/experiment/pdhd/params.jsonnet
updated: 2026-04-26
---

# source-session-2026-04-26-adc-to-electrons

Walkthrough of the signal amplitude chain from raw ADC to electrons per tick in PDHD WireCell signal processing. Focus was on how the ADC→mV conversion factor and electronics response normalization are embedded into `overall_resp`, and how the 2D deconvolution inverts this to produce charge in electrons.

## Confirmed findings

- Raw input is 14-bit ADC counts (baseline-subtracted by NF); units remain ADC counts through all NF stages — see [[ADC to Electrons Signal Chain]]
- `init_overall_response` builds `overall_resp[plane]` with units ADC·tick/e⁻ by scaling the electronics response waveform by `m_inter_gain × m_ADC_mV` before FFT (`OmnibusSigProc.cxx:853`) — see [[ADC to Electrons Signal Chain]]
- `ADC_mV_ratio = (2^14 − 1) / 1400 mV ≈ 11.70 ADC/mV` for PDHD (14-bit ADC, 1400 mV fullscale) — see [[PDHD Signal Processing Configuration]]
- PDHD FE amplifier: gain = 14 mV/fC, shaping time = 2.2 µs (`params.jsonnet`) — see [[PDHD Signal Processing Configuration]]
- `decon_2D_init` divides `m_c_data / c_resp` in 2D frequency domain; output `m_r_data` is in **e⁻ per tick** — see [[OmnibusSigProc]]
- All downstream filter passes (tight/tighter/loose, Wiener_wide, Gaus_wide) are dimensionless; they do not change units — see [[OmnibusSigProc]]
- Final `frame_wiener` and `frame_gauss` arrays: **e⁻ per tick** — see [[ADC to Electrons Signal Chain]]
- `summary_wiener`: **e⁻** (1σ noise RMS from `cal_RMS()` on tight-filtered decon) — see [[ROI Formation]]
- `postgain = 1.0` in local PDHD sp.jsonnet: no inter-stage amplification beyond the nominal electronics response — see [[PDHD Signal Processing Configuration]]
- The `(-1)` sign in `ewave` scaling ensures deconvolved charge is positive for collected electrons, consistent with the field response and ADC baseline sign convention — see [[ADC to Electrons Signal Chain]]
- Per-channel response correction (`per_chan_resp`): normalizes each channel's actual FEMB gain/shaping to the nominal before the common deconvolution (`OmnibusSigProc.cxx:1085–1101`) — see [[OmnibusSigProc]]

## Context

User asked how signal amplitude propagates through the PDHD WireCell pipeline, from incoming ADC values to the final electron count in the output arrays. `OmnibusSigProc.cxx`, `ProtoduneHD.cxx`, and the pdhd Jsonnet configs were read to trace each conversion step.
