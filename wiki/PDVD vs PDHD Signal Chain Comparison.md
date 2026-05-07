---
tags: [algorithm, concept, synthesis]
sources: 2
updated: 2026-04-26
---

# PDVD vs PDHD Signal Chain Comparison

Compares the absolute signal amplitude chain (ADC → electrons) between ProtoDUNE-VD (PDVD) and ProtoDUNE-HD (PDHD), covering electronics, ADC parameters, noise filter pipeline differences, and the resulting deconvolution sensitivity.

**Relevant files (PDVD):** `wire-cell-cfg/pgrapher/experiment/protodunevd/sp.jsonnet`, `wire-cell-cfg/pgrapher/experiment/protodunevd/params.jsonnet`, `wire-cell-toolkit/sigproc/src/OmnibusSigProc.cxx`, `wire-cell-toolkit/sigproc/src/ProtoduneVD.cxx`

**Relevant files (PDHD):** `wire-cell-cfg/pgrapher/experiment/pdhd/sp.jsonnet`, `wire-cell-cfg/pgrapher/experiment/pdhd/params.jsonnet`

## Electronics Response

| Parameter | PDHD | PDVD Bottom (ident < 4) | PDVD Top (ident ≥ 4) |
|---|---|---|---|
| Type | `ColdElecResponse` | `ColdElecResponse` | `JsonElecResponse` |
| Nominal gain | 14 mV/fC | 7.8 mV/fC | ~11 mV/fC (tabulated in file) |
| Shaping time | 2.2 µs | 2.2 µs | from file (`psnorm_400` → peak ~1300 ns) |
| `postgain` | 1.0 | 1.1365 | 1.52 |
| **Effective gain** | **14 mV/fC** | **~8.87 mV/fC** | **~16.7 mV/fC** |
| ADC fullscale | 1.4 V (0.2–1.6 V) | 1.4 V (0.2–1.6 V) | 2.0 V |
| ADC resolution | 14-bit | 14-bit | 14-bit |
| `ADC_mV_ratio` | (2¹⁴−1)/1400 ≈ **11.70 ADC/mV** | (2¹⁴−1)/1400 ≈ **11.70 ADC/mV** | (2¹⁴−1)/2000 ≈ **8.192 ADC/mV** |

`postgain` is a flat multiplicative scale applied to the entire electronics response waveform, so it is equivalent to changing the effective gain. File: `dunevd-coldbox-elecresp-top-psnorm_400.json.bz2` encodes ~11 mV/fC; comment in params.jsonnet: `// 11mV/fC, 1.94 -> 14mV/fC` shows that `postgain=1.94` would give 14 mV/fC; `postgain=1.52` is actually used.

## Effective Sensitivity (ADC counts per electron)

```
signal_per_e⁻ = gain [mV/fC] × (1/6242) [fC/e⁻] × postgain × ADC_mV_ratio [ADC/mV]
```

| | PDHD | PDVD Bottom | PDVD Top |
|---|---|---|---|
| signal per e⁻ | 14 × (1/6242) × 1.0 × 11.70 ≈ **0.02625 ADC/e⁻** | 7.8 × (1/6242) × 1.1365 × 11.70 ≈ **0.01662 ADC/e⁻** | 11 × (1/6242) × 1.52 × 8.192 ≈ **0.02197 ADC/e⁻** |

PDHD is ~58% more sensitive per electron than PDVD bottom. PDVD top is intermediate (~84% of PDHD).

## Noise Filter Pipeline Differences

| Stage | PDHD | PDVD Bottom | PDVD Top |
|---|---|---|---|
| Resampler | No | Yes (9766 @ 512 ns → 10000 @ 500 ns) | No |
| `OmnibusNoiseFilter` | Yes | Yes | Yes |
| `ShieldCouplingSub` | No | No | Yes (U plane only) |

**Resampler (PDVD bottom only):** raw HDF5 data arrives at 512 ns tick; resampled to 500 ns to match the WireCell DAQ tick. Purely structural: configured at jsonnet time by `n < 4` condition in `wcls-nf-sp.jsonnet`.

**ShieldCouplingSub (PDVD top, U plane only):** grouped filter that removes capacitive coupling noise from the top-APA shield geometry. Groups channels by strip length, computes median, subtracts, rescales back. Still operates in ADC units before deconvolution.

## Field Response Files

| | PDHD | PDVD |
|---|---|---|
| Field responses | Single file (PDHD geometry) | Two files: `fields[0]` (bottom), `fields[1]` (top) — different drift/wire geometry |

PDVD bottom and top have distinct wire geometries requiring separate field response simulations.

## Signal Processing (Deconvolution)

Both detectors use `OmnibusSigProc` with the same formula. The differences in gain, postgain, and ADC_mV_ratio are all folded into `overall_resp[plane]`:

```
overall_resp[plane] = FR × elec_resp × postgain × ADC_mV_ratio   [ADC·tick / e⁻]
decon output = FFT(ADC waveform) / FFT(overall_resp)              → e⁻ / tick
```

The per-channel response correction (`per_chan_resp`, from FEMB calibration) normalizes each channel's actual gain and shaping to the nominal before the common deconvolution — applied at `OmnibusSigProc.cxx:1085–1101`. This is present in both detectors.

## Output Scaling

Both use `frame_scale: 0.005` in the FrameSaver → stored values are in units of **200 electrons** (multiply stored value by 200 to get electrons/tick).

## Tick count through pipeline

| Stage | PDHD | PDVD Bottom | PDVD Top |
|---|---|---|---|
| Raw HDF5 | uniform | 9766 @ 512 ns | 10048 @ 512 ns (or 10000 for some events) |
| After Resampler | — | 10000 @ 500 ns | (no resampler) |
| After NF/SP | 10000 | 10000 | 10048 (or 10000) |

## Summary of Key Differences

| Aspect | PDHD | PDVD |
|---|---|---|
| Electronics uniformity | Single type, all channels | Two types: bottom (`ColdElec` 7.8 mV/fC) vs top (`JsonElecResponse` ~11 mV/fC) |
| Effective gain | 14 mV/fC | 8.87 mV/fC (bottom), ~16.7 mV/fC (top) |
| ADC dynamic range | 1.4 V fullscale everywhere | 1.4 V bottom, 2.0 V top |
| ADC sensitivity | 11.70 ADC/mV | 11.70 ADC/mV (bottom), 8.192 ADC/mV (top) |
| Tick rate mismatch | No | Yes: bottom raw @ 512 ns → needs Resampler |
| Extra NF stage | None | `ShieldCouplingSub` for top U plane |
| Field responses | One | Two (separate bottom/top geometry) |
| ADC/e⁻ sensitivity | 0.02625 (highest) | 0.01662 bottom (lowest), 0.02197 top |

## See also

- [[ADC to Electrons Signal Chain]] (PDHD detailed chain)
- [[PDVD Signal Processing Configuration]]
- [[PDHD Signal Processing Configuration]]
- [[OmnibusSigProc]]

## Sources

- [[source-session-2026-04-26-adc-to-electrons]] (PDHD baseline)
- [[source-session-2026-04-26-pdvd-signal-chain]]
