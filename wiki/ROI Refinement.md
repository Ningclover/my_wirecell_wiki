---
tags: [algorithm]
sources: 1
updated: 2026-04-14
---

# ROI Refinement

Multi-stage iterative cleanup of ROIs identified by [[ROI Formation]]. The largest file in `sigproc` (3,155 lines).

**File:** `src/ROI_refinement.cxx`  
**Header:** `src/ROI_refinement.h`

## Purpose

Remove fake signal ROIs while preserving real ones. Uses multi-plane geometric consistency and peak-finding to produce clean, well-bounded regions for final filtering.

## Relationship maps maintained

- `front_rois[roi]` — ROIs on the next wire that overlap in time
- `back_rois[roi]` — ROIs on the previous wire that overlap in time
- `contained_rois[loose_roi]` — tight ROIs contained within a loose ROI

## Pipeline stages

### 1. CleanUpROIs
BFS flood-fill from tight ROIs. Any loose ROI not reachable (directly or transitively) from a tight ROI is removed.

### 2. generate_merge_ROIs
If a tight ROI is not contained in any loose ROI, promote it (create a new loose ROI to cover it).

### 3. MP3ROI / MP2ROI
Multi-plane geometric protection: signal regions must be consistent across 3 (or 2) wire planes. Removes ROIs that don't have corresponding signal in other planes.

### 4. BreakROIs
TSpectrum-like peak finding (via [[PeakFinding]]) to split wide ROIs at valleys. Runs for `m_break_roi_loop` iterations (default 2).

### 5. ShrinkROIs
Tighten ROI boundaries: if neighbor ROIs don't support the current extent, shrink to the actual signal region.

### 6. CleanUpCollectionROIs / CleanUpInductionROIs
Remove likely-fake signals based on charge ratio and shape criteria. Separate logic for collection vs. induction planes.

### 7. ExtendROIs
Extend each ROI to cover the time range of overlapping neighbor ROIs. Ensures consistent coverage for baseline subtraction.

## Known issues / bugs

- **BUG-CORE-1** (FIXED): Division by zero in `apply_roi` when `start_bin == end_bin`.
- **BUG-CORE-2** (FIXED): Division by zero in `BreakROI` when two consecutive valley positions coincide.
- **BUG-CORE-4** (NOT FIXED): Float-to-int truncation in baseline subtraction (up to 1 ADC count).
- **BUG-CORE-6** (FIXED): Unsigned underflow in `section_protected` — `saved_b.size()-1` wraps to UINT_MAX when `saved_b` is empty. Fixed by casting to `int`.
- **BUG-CORE-7** (NOT FIXED): Fixed-size stack array `float valley_pos[205]` in `BreakROI`. If `max_npeaks` configured > 200, stack overflow.
- **BUG-CORE-8** (FIXED): Float comparison for integer wrap boundary (`nwire_w / 2.` used float division). Changed to integer division.
- **BUG-CORE-10** (NOT FIXED, dead code): `TestROIs` boundary access would OOB, but function not called in production.
- **EFF-CORE-2** (NOT FIXED): 3× code duplication for U/V/W planes.
- **EFF-CORE-3** (FIXED): Cached `bad_ch_map` lookups in inner loops.
- **EFF-CORE-5** (FIXED): `get_mp2_rois()` non-const overload now returns by reference instead of copying the entire map.
- **EFF-CORE-7**: `SignalROI` objects managed with raw `new`/`delete` — risk of leaks on exception paths.

## See also

[[OmnibusSigProc]], [[ROI Formation]], [[PeakFinding]]

## Sources

- [[source-sigproc-examination]]
