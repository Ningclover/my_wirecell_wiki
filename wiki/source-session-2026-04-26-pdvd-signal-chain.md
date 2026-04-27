---
tags: [source]
type: conversation
date: 2026-04-26
context: Tracing the absolute signal amplitude through the full PDVD WireCell pipeline — raw ADC → NF → 2D deconvolution → electrons/tick, with separate analysis for bottom and top APAs, and comparison to PDHD
files_touched:
  - wire-cell-toolkit/sigproc/src/OmnibusSigProc.cxx
  - wire-cell-toolkit/sigproc/src/ProtoduneVD.cxx
  - wire-cell-cfg/pgrapher/experiment/protodunevd/sp.jsonnet
  - wire-cell-cfg/pgrapher/experiment/protodunevd/params.jsonnet
  - wire-cell-cfg/dunevd-coldbox-elecresp-top-psnorm_400.json.bz2
updated: 2026-04-26
---

# source-session-2026-04-26-pdvd-signal-chain

Walkthrough of the PDVD signal amplitude chain from raw ADC to electrons per tick, separately for bottom (ident < 4) and top (ident ≥ 4) APAs. Followed by comparison to PDHD chain documented in [[ADC to Electrons Signal Chain]].

## Confirmed findings

- Bottom uses `ColdElecResponse` (7.8 mV/fC, shaping 2.2 µs, postgain 1.1365); top uses `JsonElecResponse` from `dunevd-coldbox-elecresp-top-psnorm_400.json.bz2` (~11 mV/fC peak, postgain 1.52) — see `params.jsonnet`
- `postgain` is a flat multiplier on the entire response waveform and is equivalent to rescaling the effective gain: top effective gain ≈ 11 × 1.52 ≈ 16.7 mV/fC
- ADC fullscale differs: 1.4 V for bottom (same as PDHD), 2.0 V for top → `ADC_mV_ratio` = 11.70 ADC/mV (bottom) vs 8.192 ADC/mV (top) — see `sp.jsonnet`
- Resulting ADC/e⁻ sensitivity: 0.01662 (bottom, lowest), 0.02197 (top), 0.02625 (PDHD, highest)
- PDVD bottom requires a Resampler (9766 @ 512 ns → 10000 @ 500 ns); configured structurally at jsonnet time by `n < 4` condition
- PDVD top U plane has `ShieldCouplingSub` extra NF stage (not present in bottom or PDHD); operates in ADC units before deconvolution
- PDVD has two separate field response files (bottom geometry vs top geometry); PDHD has one
- `JsonElecResponse.cxx`: file amplitudes are loaded with no unit conversion and interpolated onto WireCell tick grid; `postgain` applied after loading
- All downstream: same `OmnibusSigProc` deconvolution formula as PDHD; differences absorbed into `overall_resp`; output is e⁻/tick
- Both PDVD and PDHD use `frame_scale: 0.005` → stored in units of 200 electrons

## Context

User asked to trace the absolute signal value from raw ADC to electrons across the PDVD pipeline, focusing on the difference between bottom and top APAs. Then asked to compare to the PDHD wiki page [[ADC to Electrons Signal Chain]]. The `JsonElecResponse` amplitude units were investigated: the file stores amplitudes in an internal WireCell unit such that after `waveform_samples()` returns values and OmnibusSigProc scales by `m_inter_gain * m_ADC_mV`, the product has units ADC·tick/e⁻ consistent with the PDHD chain. The numeric value 1.16e-12 at peak is in WireCell native units (not directly mV/fC as a human-readable number).
