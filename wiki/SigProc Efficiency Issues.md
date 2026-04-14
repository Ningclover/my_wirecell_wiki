---
tags: [meta]
sources: 1
updated: 2026-04-14
---

# SigProc Efficiency Issues

Consolidated efficiency recommendations from systematic examination. **11 fixed, 14 not fixed** out of 25 total.

## High Impact (4 issues — 3 FIXED)

| ID | Location | Fixed? | Description |
|----|----------|--------|-------------|
| EFF-CORE-1 | `ROI_formation.cxx:364` | FIXED | `cal_RMS` took full waveform by value (~10K floats × thousands of calls per frame). Changed to `const&`. |
| EFF-L1-1 | `L1SPFilter.cxx:473` | FIXED | ~20 JSON `get()` calls per `L1_fit` invocation. All values now cached in `configure()`. |
| EFF-L1-5 | `L1SPFilter.cxx:496` | FIXED | Smearing vector re-parsed from JSON per call. Now cached in `configure()`. |
| EFF-CORE-2 | `ROI_formation.cxx`, `ROI_refinement.cxx` | NOT FIXED | 3× code duplication for U/V/W planes. Large-scale refactor needed — plane-parametrized helper. |

## Medium Impact (8 issues — 5 FIXED)

| ID | Location | Fixed? | Description |
|----|----------|--------|-------------|
| EFF-NF-2 | `OmnibusNoiseFilter.cxx` | FIXED | Range-for loops copying `shared_ptr`s, vectors, map entries. Changed to `const auto&`. |
| EFF-NF-1 | `OmnibusPMTNoiseFilter.cxx` | FIXED | Double waveform copy per trace. Eliminated. |
| EFF-NF-7 | `OmnibusNoiseFilter.cxx:185` | FIXED | O(N) linear search for bad channels. Replaced with `unordered_set`. |
| EFF-CORE-3 | `ROI_refinement.cxx:291`, `ROI_formation.cxx:411` | FIXED | `bad_ch_map` looked up 3× per column in inner loops. Now cached. |
| EFF-NF-3 | `OmnibusNoiseFilter.cxx:229` | Partial | Grouped filter copies waveforms in/out of map. Writeback now uses `auto&`; initial copy required by filter API. |
| EFF-CORE-4 | `OmnibusSigProc.cxx` | NOT FIXED | `c_data_afterfilter` full complex array allocated ~8× per plane. Workspace member would help. |
| EFF-L1-2 | `L1SPFilter.cxx:564` | NOT FIXED | Dense matrix (up to 120×240, ~230 KB) allocated per LASSO segment. Pre-allocate scratch buffer. |
| EFF-DET-1 | All detector files | NOT FIXED | Per-channel FFT inherent to algorithm; DFT plans likely cached internally. |
| EFF-DET-2 | All detector files | NOT FIXED | `auto temp_signal = signal` waveform copy for sigma-clipping. Histogram approach would avoid it. |

## Low Impact (13 issues — 3 FIXED)

| ID | Location | Fixed? | Description |
|----|----------|--------|-------------|
| EFF-CORE-5 | `ROI_refinement.h:47` | FIXED | `get_mp2_rois()` non-const returned by value. Now returns by reference. |
| EFF-DET-3 | All detector files | FIXED | `CalcRMSWithFlags` `push_back` without `reserve()`. Added `reserve(sig.size())`. |
| EFF-NF-4 | `OmniChannelNoiseDB.cxx:131` | NOT FIXED | `parse_channels` iterates all anode channels per JSON entry. |
| EFF-DET-4 | All detector files | NOT FIXED | ROIs iterated by value: `for (auto roi : rois)`. |
| EFF-DET-5 | `ProtoduneVD.cxx:1159` | NOT FIXED | Per-bin temp vector in shield coupling median. |
| EFF-DET-6 | `ProtoduneVD.cxx:1255` | NOT FIXED | `ShieldCouplingSub` makes 3 full passes; could be fewer. |
| EFF-UTIL-* | Various | NOT FIXED | `PeakFinding` raw `new[]`, `ChannelSelector` uses `std::set`, `CalcMedian` per-bin alloc, etc. |
| EFF-CORE-6 | `OmnibusSigProc.cxx` | NOT FIXED | Filter objects re-fetched via `Factory::find` on every invocation. |
| EFF-CORE-7 | `ROI_refinement.cxx` | NOT FIXED | `SignalROI` objects managed with raw `new`/`delete` — unsafe on exception paths. |

## See also

[[SigProc Bug List]], [[WireCellSigProc Pipeline Overview]]

## Sources

- [[source-sigproc-examination]]
