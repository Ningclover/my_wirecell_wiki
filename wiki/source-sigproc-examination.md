---
tags: [source]
updated: 2026-04-14
---

# Source: WireCellSigProc Code Examination

Systematic examination of the `sigproc` module (~16,600 lines of C++ across 30 source files).

## What was examined

- Potential bugs: logic errors, off-by-one, division-by-zero, numeric pitfalls
- Running efficiency: unnecessary copies, redundant computation, memory allocation patterns
- Algorithm documentation: what each component does and how it fits the pipeline

## Files covered

| Group | Files | Lines |
|-------|-------|-------|
| Core SP pipeline | `OmnibusSigProc.cxx`, `ROI_refinement.cxx`, `ROI_formation.cxx` | ~6,000 |
| L1SP filter | `L1SPFilter.cxx` | 692 |
| Noise filtering | `OmnibusNoiseFilter`, `OmnibusPMTNoiseFilter`, `OmniChannelNoiseDB`, `SimpleChannelNoiseDB`, `NoiseModeler` | ~2,000 |
| Detector-specific | `Microboone`, `Protodune`, `ProtoduneHD`, `ProtoduneVD`, `DuneCrp`, `Icarus` | ~5,100 |
| Utilities | `PeakFinding`, `ChannelSelector`, `FrameMerger`, `Diagnostics`, filters, etc. | ~1,800 |

## Examination documents

Located at `raw/wire-cell-toolkit/sigproc/docs/examination/`:

- `00-plan.md` — examination plan
- `01-overview.md` — pipeline architecture overview
- `02-core-sigproc.md` — OmnibusSigProc, ROI formation/refinement
- `03-l1sp-filter.md` — L1SP filter algorithm
- `04-noise-filtering.md` — noise filtering subsystem
- `05-detector-specific.md` — per-detector implementations
- `06-utilities.md` — utility components
- `07-bug-summary.md` — consolidated bug list (40 bugs, 17 fixed)
- `08-efficiency-summary.md` — consolidated efficiency issues (25 issues, 11 fixed)

## Fix summary

- **17/40 bugs fixed** — all 5 critical/high severity bugs addressed
- **11/25 efficiency issues fixed** — all 3 high-impact issues addressed
- Remaining unfixed items require physics expert input or large-scale refactoring

## See also

[[WireCellSigProc Pipeline Overview]], [[OmnibusSigProc]], [[ROI Formation]], [[ROI Refinement]], [[L1SP Filter]], [[Omnibus Noise Filter]], [[Detector-Specific Signal Processing]], [[SigProc Bug List]], [[SigProc Efficiency Issues]]
