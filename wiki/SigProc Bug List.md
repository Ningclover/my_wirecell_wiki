---
tags: [meta]
sources: 1
updated: 2026-04-14
---

# SigProc Bug List

Consolidated bug list from systematic examination of `sigproc`. **17 fixed, 23 not fixed** out of 40 total.

## Critical / High (5 bugs — all FIXED)

| ID | Location | Description |
|----|----------|-------------|
| BUG-NF-1 | `OmnibusPMTNoiseFilter.cxx:23` | Global mutable `PMT_ROIs` vector — not thread-safe, leaks on exception. Moved to member. |
| BUG-NF-2 | `OmnibusPMTNoiseFilter.cxx:265` | `min` never updated in PMT peak loop — reports wrong peak position. |
| BUG-L1-1 | `L1SPFilter.cxx:378-397` | Reverse-scan uses `size-1-i1` for distance but forward `i1` for flag lookup — entire reverse pass operates on wrong ROIs. |
| BUG-DET-1 | `Protodune.cxx:343-346` | `min`/`max` iterator names swapped in `LinearInterpSticky` — inverts signal-like detection logic. |
| BUG-UTIL-1 | `PeakFinding.h:35-39` | Raw pointers uninitialized (`nullptr`). Destructor crashes if `find_peak()` never called. Memory leak on repeated calls. |

## Medium (18 bugs — 12 FIXED, 6 NOT FIXED)

| ID | Location | Fixed? | Description |
|----|----------|--------|-------------|
| BUG-CORE-1 | `ROI_refinement.cxx:139`, `ROI_formation.cxx:76` | FIXED | Div/0 in `apply_roi` when `start_bin == end_bin` |
| BUG-CORE-2 | `ROI_refinement.cxx:2156` | FIXED | Div/0 in `BreakROI` when valley positions coincide |
| BUG-CORE-3 | `ROI_formation.cxx:865` | FIXED | Dead merge loop — `flag_repeat=0` before `while(flag_repeat)` |
| BUG-CORE-6 | `ROI_refinement.cxx:2265` | FIXED | Unsigned underflow: `saved_b.size()-1` wraps to UINT_MAX |
| BUG-L1-2 | `L1SPFilter.cxx:527` | FIXED | Div/0 in flag classification — `&&` short-circuit doesn't protect left operand |
| BUG-NF-3 | `OmnibusPMTNoiseFilter.cxx:393` | FIXED | Div/0 in `RemovePMTSignal` when `end_bin == start_bin` |
| BUG-DET-2 | `Microboone.cxx:722`, `PDHD:620`, `PDVD:621`, `DuneCrp:184` | FIXED | Div/0 in `RawAdapativeBaselineAlg` when `upIndex == downIndex` |
| BUG-UTIL-2 | `Diagnostics.cxx:76` | FIXED | NaN in chirp RMS from `sqrt()` of slightly negative float |
| BUG-UTIL-3 | `Diagnostics.cxx:17` | FIXED | Partial spectrum OOB: `spec[ind+1]` not fully guarded |
| BUG-UTIL-5 | `Undigitizer.cxx:89` | NOT FIXED | ADC max off-by-one: `1<<bits` should be `(1<<bits)-1` |
| BUG-UTIL-6 | `Undigitizer.cxx:104` | NOT FIXED | `baselines[plane]` unchecked |
| BUG-UTIL-7 | `ChannelSelector.cxx:120` | NOT FIXED | `summary[trind]` may exceed summary size |
| BUG-UTIL-8 | `ChannelSplitter.cxx:105` | NOT FIXED | Input frame trace indices passed to output frame (invalid) |
| BUG-NF-4 | `OmniChannelNoiseDB.cxx:238` | NOT FIXED | Reconfig cache key overflow (8-bit fields, values > 25.5 overflow) |
| BUG-NF-5 | `OmniChannelNoiseDB.cxx:661` | NOT FIXED | Empty dummy filter returned instead of properly-sized filter |
| BUG-NF-6 | `OmnibusNoiseFilter.cxx:226` | NOT FIXED | Entire group skipped for coherent noise if one channel missing |
| BUG-DET-3 | `Protodune.cxx:321` | NOT FIXED | Hardcoded 6000-tick count in legacy `LedgeIdentify` |
| BUG-DET-4 | `Protodune.cxx:890` | NOT FIXED | ProtoDUNE-SP uses MicroBooNE 12-bit (4096) ADC threshold — needs hardware confirmation |
| BUG-UTIL-4 | `FrameMerger.cxx:93` | NOT FIXED | Unknown merge rule silently produces empty output |

## Low (17 bugs — 2 FIXED, 15 NOT FIXED)

| ID | Fixed? | Description |
|----|--------|-------------|
| BUG-CORE-4 | NOT FIXED | Float-to-int truncation in baseline subtraction (up to 1 ADC count) |
| BUG-CORE-5 | FIXED | Config key typos: `"r_th_precent"`, `"isWarped"` |
| BUG-CORE-7 | NOT FIXED | Fixed-size stack array `[205]` vs configurable `max_npeaks` in `BreakROI` |
| BUG-CORE-8 | FIXED | Float comparison for integer wrap boundary |
| BUG-CORE-9 | NOT FIXED | Possibly inverted interpolation weights in response redigitization |
| BUG-CORE-10 | NOT FIXED (dead code) | `TestROIs` OOB — function not called in production |
| BUG-L1-3..6 | NOT FIXED | Various L1SP minor issues (see [[L1SP Filter]]) |
| BUG-NF-7..11 | NOT FIXED | Various NoiseDB/Modeler minor issues (see [[Omnibus Noise Filter]], [[OmniChannelNoiseDB]]) |
| BUG-DET-5..9 | NOT FIXED | Various detector-specific minor issues (see [[Detector-Specific Signal Processing]]) |
| BUG-UTIL-9..11 | NOT FIXED | Various utility minor issues |

## Files modified

`OmnibusPMTNoiseFilter`, `L1SPFilter`, `Protodune`, `PeakFinding`, `ROI_formation`, `ROI_refinement`, `OmnibusSigProc`, `Diagnostics`, `Microboone`, `ProtoduneHD`, `ProtoduneVD`, `DuneCrp`

## See also

[[SigProc Efficiency Issues]], [[WireCellSigProc Pipeline Overview]]

## Sources

- [[source-sigproc-examination]]
