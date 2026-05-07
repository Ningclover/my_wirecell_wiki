---
tags: [synthesis]
sources: 1
updated: 2026-05-07
---

# Img Efficiency Issues

15 efficiency issues catalogued from systematic examination of `wirecell-working/toolkit/img/`.
7 fixed, 8 not fixed.

## Algorithmic Complexity (Not Fixed — Require Redesign)

| # | File | Issue | Complexity | Status |
|---|------|-------|------------|--------|
| 1 | `InSliceDeghosting.cxx:362-516` | Pairwise 2-view vs 3-view adjacency + overlap scoring per slice | O(n_2view × n_3view × n_channels) | NOT FIXED |
| 2 | `ProjectionDeghosting.cxx` | Pairwise projection coverage comparison for all cluster pairs per plane | O(n_clusters²) | NOT FIXED |
| 3 | `CMMModifier.cxx:189-252` | Triple nested loop: channels × adjacent channels × time range alignment | O(n³) | NOT FIXED |
| 4 | `BlobDepoFill.cxx:293-366` | Brute-force slice × depo × wire × blob intersection checking | O(n⁴) worst case | NOT FIXED |
| 5 | `FrameQualityTagging.cxx:319-363` | Unbounded `while` loops searching for signal edges per channel | Unbounded | NOT FIXED |

## Redundant Computations

| # | File | Issue | Status |
|---|------|-------|--------|
| 6 | `Projection2D.cxx:469-480` | 7 separate sparse matrix passes (count+sum) where 3 suffice | **FIXED**: 3 single-pass loops each computing all stats |
| 7 | `FrameQualityTagging.cxx:420-434` | Global fire rate rebinning repeats work already done in per-plane analysis | NOT FIXED |
| 8 | `BlobSetReframer.cxx:86-101` | Per-miss cache rebuild iterated all planes each time; multiple misses = repeated scans | **FIXED**: `ensure_cache()` pre-populates once on first use |
| 9 | `ChargeSolving.cxx:268` | `dump_cg()` traversed entire cluster graph unconditionally at all log levels | **FIXED**: guarded with `log->level() <= debug` |

## Unnecessary Copies and Allocations

| # | File | Issue | Status |
|---|------|-------|--------|
| 10 | `CSGraph.cxx:160` | Explicit O(n³) matrix inverse then Cholesky — redundant and unstable | **FIXED**: direct Cholesky + triangular solve; eliminates explicit inverse |
| 11 | `Projection2D.cxx:297` | Dense conversion of entire sparse matrix for file I/O | NOT FIXED (diagnostic path, Stream API limitation) |
| 12 | `GlobalGeomClustering.cxx` | `dump_cg`, 2× `connected_components`, 4× debug log messages ran unconditionally | **FIXED**: guarded with `log->level() <= debug` |
| 13 | `BlobGrouping.cxx:55-109` | 3 subgraphs built per slice via multiple iterations over adjacent vertices | NOT FIXED (savings marginal; single-pass would be minor improvement) |
| 14 | `MaskSlice.cxx:303-304` | `std::find` on vector for plane membership check in per-trace hot loop | **FIXED**: precomputed `std::unordered_set<int>` → O(1) lookup |
| 15 | `InSliceDeghosting.cxx:259-265` | `std::set_intersection` into temp vector just to count common elements (per blob pair, tight inner loop) | **FIXED**: replaced with manual merge-count loop (same O(n+m), no allocation) |

## Summary

| Category | Fixed | Not Fixed |
|----------|-------|-----------|
| Algorithmic complexity | 0 | 5 |
| Redundant computation | 3 | 1 |
| Copies / allocation | 4 | 3 |
| **Total** | **7** | **8** |

## Hardcoded Constants Needing Review for PDHD

| Priority | Constant | Value | Location |
|----------|----------|-------|----------|
| HIGH | `MaskSlice.default_threshold` | UBooNE RMS×4 | `inc/WireCellImg/MaskSlice.h:77-79` |
| HIGH | `FrameQualityTagging` time window | 3180–7870 ticks | `src/FrameQualityTagging.cxx:49-68` |
| HIGH | `good_blob_charge_th` | 300 | `ChargeSolving` + `InSliceDeghosting` |
| MEDIUM | LASSO λ/tolerance formula factors | 3, 2, 0.005 | `src/CSGraph.cxx:144-145` |
| MEDIUM | Blob weight constants | 9, 3 | `src/ChargeSolving.cxx:124-130` |
| MEDIUM | Dead-time clustering offset | 4×500µs | `src/GeomClusteringUtil.cxx:34` |
| LOW | Sentinel values | ±1e12 | Throughout `Projection2D` |
| LOW | 3-plane assumption | `bcs(3)` | Multiple files |

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[Img Bug List]]
- [[Slicing and Tiling]]
- [[Charge Solving]]
- [[Imaging Deghosting]]

## Sources

- [[source-img-examination]]
