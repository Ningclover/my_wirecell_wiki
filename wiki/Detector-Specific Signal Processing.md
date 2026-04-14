---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# Detector-Specific Signal Processing

Each supported detector has its own noise filtering functions registered as WCT components. They share a common pipeline skeleton but differ in the noise types they handle.

## Common per-channel pipeline

All detectors follow the same general order:
1. Nominal baseline subtraction
2. Gain correction
3. Detector-specific preprocessing (chirp, sticky codes, FEMB issues, etc.)
4. FFT → frequency domain
5. RC undershoot correction (spectral division)
6. Spectral noise mask application
7. DC removal, IFFT
8. Robust baseline correction (sigma-clipping + median)
9. Adaptive baseline for partial RC channels
10. Noisy channel tagging (RMS-based)

## Flag encoding system

ADC values are overloaded to carry status flags:
- **12-bit detectors** (MicroBooNE, Protodune-SP): signal flag = 4096, dead = 10000, protected = +20000
- **14-bit detectors** (PDHD, PDVD, DuneCrp): signal flag = 16384, dead = 100000, protected = +200000

## Feature comparison

| Feature | MicroBooNE | ProtoDUNE-SP | PDHD | PDVD | DuneCrp | ICARUS |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| ADC bits | 12 | 12 | 14 | 14 | 14 | — |
| Sticky code mitigation | — | Yes | — | — | — | — |
| Chirp detection | Yes | — | — | — | — | — |
| Harmonic noise filter | — | Yes | — | — | — | — |
| RC undershoot correction | Yes | Yes | commented out | disabled | Yes | Yes |
| Adaptive baseline window | 20 | 20 | 512 | 512 | 512 | — |
| Coherent noise subtraction | Yes | — | Yes | Yes | — | — |
| FEMB noise detection | — | — | Yes | Yes | — | — |
| Shield coupling subtraction | — | — | — | Yes | — | — |
| ADC bit shift correction | Yes | — | — | — | — | — |
| Relative gain calibration | — | Yes | — | — | — | — |

## MicroBooNE

Foundational implementation; most other detectors derive from it.

**Unique features:**
- ADC bit shift detection/correction (lower bits stuck or shifted)
- Chirp noise detection (`Diagnostics::Chirp`): window-based RMS analysis
- Partial RC undershoot detection (`Diagnostics::Partial`): spectral shape
- Low-frequency noise identification (`OneChannelStatus::ID_lf_noisy`)

## ProtoDUNE-SP

**Files:** `src/Protodune.cxx` (1,020 lines)

**Unique features:**
- **Sticky code mitigation** (two-stage):
  1. Linear interpolation for non-signal-like sticky regions
  2. FFT-based interpolation using even/odd subsample prediction
- **Ledge artifact removal**: step-function + exponential recovery detection
- **50 kHz harmonic removal**: 5 iterative passes of frequency-domain outlier detection
- **FEMB clock correction** (`FftScaling`): frequency-domain resampling
- **Relative gain calibration**: pulse-area-based from JSON

## ProtoDUNE-HD (PDHD)

**Files:** `src/ProtoduneHD.cxx` (1,043 lines)

**Unique features:**
- **FEMB negative pulse detection** (`FEMBNoiseSub`): projects all channels in a group to 1D, finds wide ROIs below -3.5σ
- Wide adaptive baseline: 512-tick window (vs. 20 for MicroBooNE)

## ProtoDUNE-VD (PDVD)

**Files:** `src/ProtoduneVD.cxx` (1,354 lines)

**Unique features:**
- **Shield coupling subtraction** (novel):
  1. Scale each channel by inverse strip length (capacitance weighting)
  2. Positive-only signal protection with wide padding (70 bins)
  3. Compute median excluding flagged/outlier bins
  4. Subtract median (fixed scaling = 1)
  5. Scale back by strip length

## DUNE CRP

**Files:** `src/DuneCrp.cxx` (369 lines) — clean, minimal implementation.

## ICARUS

**Files:** `src/Icarus.cxx` (110 lines) — simplest; RC undershoot only via `IWaveform`.

## Known issues / bugs

- **BUG-DET-1** (FIXED): `LinearInterpSticky` in `Protodune.cxx` — `min`/`max` iterator names swapped. `min` pointed to max element and vice versa, inverting signal-like detection logic.
- **BUG-DET-2** (FIXED): Division by zero in `RawAdapativeBaselineAlg` when `upIndex == downIndex`. Fixed in all 4 files.
- **BUG-DET-3** (NOT FIXED): Legacy `LedgeIdentify` hardcodes 6000 ticks; newer `LedgeIdentify1` uses `signal.size()`.
- **BUG-DET-4** (NOT FIXED): ProtoDUNE-SP calls MicroBooNE functions using 4096 flag threshold. If ProtoDUNE-SP has 14-bit ADCs, values above 4096 would be misclassified. Needs hardware expert confirmation.
- **BUG-DET-5** (NOT FIXED): Protodune harmonic removal — `spectrum.at(mag.size()+1-j)` OOB when j==1.
- **BUG-DET-6** (NOT FIXED): PDHD/PDVD `Subtract_WScaling` hardcodes `correlation_threshold=4` instead of using configurable parameter.
- **BUG-DET-7** (NOT FIXED): PDVD `Subtract_WScaling` uses `ave_coef > 0` instead of `!= 0`.
- **BUG-DET-8** (NOT FIXED): Static `warned` flag in Protodune.cxx — only first mismatch ever logged.
- **EFF-DET-3** (FIXED): `CalcRMSWithFlags` now pre-reserves vector to avoid reallocation.

## See also

[[WireCellSigProc Pipeline Overview]], [[Omnibus Noise Filter]], [[MicroBooNE]], [[ProtoDUNE-SP]], [[ProtoDUNE-HD]], [[ProtoDUNE-VD]]

## Sources

- [[source-sigproc-examination]]
