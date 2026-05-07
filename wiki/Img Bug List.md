---
tags: [synthesis]
sources: 1
updated: 2026-05-07
---

# Img Bug List

18 bugs catalogued from systematic examination of `wirecell-working/toolkit/img/`.
13 fixed, 5 unfixed (pre-existing limitations).

## HIGH Severity

| # | File | Bug | Status |
|---|------|-----|--------|
| 1 | `InSliceDeghosting.cxx:66` | `pack &= (0 << p)` always clears ALL bits — should be `pack &= ~(1 << p)` | **FIXED** |
| 2 | `InSliceDeghosting.cxx:264` | Division by zero when `wires1.size() == 0` (2-view blob on dead plane) | **FIXED** |
| 3 | `CSGraph.cxx:160` | No check for singular `mcov` — Cholesky failure produces NaN/Inf charges silently | **FIXED** |
| 4 | `CSGraph.cxx:94-104` | Single-blob path didn't apply `params.scale` and used uninitialized `val` | **FIXED** |
| 5 | `Projection2D.cxx:420` | Unreachable `return OTHER` after complete if-else chain | **FIXED** |

## MEDIUM Severity

| # | File | Bug | Status |
|---|------|-----|--------|
| 6 | `Projection2D.cxx:517-520` | Division by zero if `small_charge==0 or small_counts==0` in `judge_coverage_alt` | **FIXED** |
| 7 | `BlobGrouping.cxx:52` | Hard-coded `bcs(3)` — breaks for non-3-plane detectors | NOT FIXED (pre-existing) |
| 8 | `DeadLiveMerging.cxx:82-83` | "Dummy implementation", FIXME on ident — incomplete merge logic | NOT FIXED (pre-existing) |
| 9 | `GeomClusteringUtil.cxx:26-28` | `%d` format for `double` values; error msg said "tmax" but printed `tmin` | **FIXED** |
| 10 | `Projection2D.cxx:228-237` | Floating-point `==` comparison with sentinel 1e12; zero charge conflated with no data | **FIXED** |
| 11 | `ChargeSolving.cxx:114-115` | `good_blob_charge_th=300` hardcoded with TODO; should match `InSliceDeghosting` config | **FIXED** |
| 12 | `BlobDeclustering.cxx:51` | `nullptr` passed as slice to `SimpleBlobSet` — potential null deref downstream | **FIXED** |

## LOW Severity

| # | File | Bug | Status |
|---|------|-----|--------|
| 13 | `Projection2D.cxx` | Inconsistent dead-pixel criteria: `judge_coverage` counts `> -uncer_cut` as live; `judge_coverage_alt` counts only `> 0` | NOT FIXED (likely intentional) |
| 14 | `Projection2D.cxx:126-134` | Weak XOR hash `h1^h2` for (channel,slice) pairs — symmetric collisions | **FIXED** (boost hash_combine) |
| 15 | `ChargeSolving.cxx:104` | `(int)` cast on float slice time causes precision loss in comparisons | **FIXED** |
| 16 | `ChargeSolving.cxx` | `dump_cg()` / `dump_sg()` iterate full graph unconditionally at production log levels | **FIXED** |
| 17 | `FrameQualityTagging.cxx:83` | Copy-paste: `m_n_fire_cut2` default used `m_n_cover_cut2` value | **FIXED** |
| 18 | `ShadowGhosting.cxx:49` | `m_shadow_type[0]` accessed without empty-string check — UB if empty | **FIXED** |

## Summary

- **FIXED**: 13/18
- **NOT FIXED**: 5/18 (3 pre-existing limitations, 1 likely intentional, 1 unknown risk)

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[Imaging Deghosting]]
- [[Charge Solving]]
- [[Img Efficiency Issues]]

## Sources

- [[source-img-examination]]
