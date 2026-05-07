---
tags: [algorithm]
sources: 1
updated: 2026-05-07
---

# Imaging Deghosting

Removes spurious 3D "ghost" blobs from the imaging result.
Part of the [[WireCell Imaging Pipeline Overview]].

## Background: What Are Ghosts?

In a 3-plane LArTPC, each plane provides a 1D projection of the charge distribution.
A blob is the 3D intersection of active wire strips from all planes.
"Ghosts" arise when wires from unrelated real deposits happen to cross, creating
false 3D intersections. Deghosting removes these artifacts.

---

## InSliceDeghosting (`src/InSliceDeghosting.cxx`)

### Purpose

Local, within-slice ghost removal based on wire scores and blob adjacency.
Runs immediately after initial clustering, before charge solving (or uses
existing charge values if run after).

### Bit-tag System

Each blob carries a packed integer tag (enum `BLOB_QUALITY_BITPOS`):

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | GOOD | Charge above threshold |
| 1 | BAD | — |
| 2 | POTENTIAL_GOOD | Neighbor has good charge |
| 3 | POTENTIAL_BAD | — |
| 4 | TO_BE_REMOVED | Marked for deletion |

> **Bug fixed**: `pack &= (0 << p)` always zeroed all bits; corrected to `pack &= ~(1 << p)`.

### Phase 1: Blob Quality (`blob_quality_ident`)

- Blob charge > `m_good_blob_charge_th` (default 300) → tag GOOD + POTENTIAL_GOOD
- If any b-b neighbor has charge > threshold → tag current blob POTENTIAL_GOOD

### Phase 2: Local Deghosting (`local_deghosting1`)

Per slice:
1. **Group blobs** by view count: `view_groups[3]` (3-view) and `view_groups[2]` (2-view)
2. **Wire score map**: count how many 3-view blobs share each wire; high score = ambiguous wire
3. **Protected blobs**: a 2-view blob is protected if `adjacent()` to ≥2 POTENTIAL_GOOD 3-view blobs

   Adjacency test (per plane, score summed across planes ≥5):
   - Score 2: channels overlap
   - Score 1: channels are adjacent (differ by ±1)
   - Score 0: neither → immediate fail for that plane

4. **Score-based removal**: for each 2-view blob:
   - Compute `blob_high_score_map` (max across planes of average 1/wire_score)
   - For overlapping blobs with higher score: check `overlap_ratio ≥ m_deghost_th` AND `q2 > q1 × m_deghost_th`
   - If 2 such blobs found AND blob not protected → tag TO_BE_REMOVED

### Phase 3: Graph Filtering

Remove TO_BE_REMOVED blobs from cluster graph. Optionally re-cluster with `geom_clustering()`.

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `m_good_blob_charge_th` | 300 | "Good" blob charge threshold |
| `m_deghost_th` | 0.75 | Wire overlap ratio threshold |
| `m_deghost_th1` | 0.5 | Secondary deghosting threshold |

### Complexity

O(n_2view × n_3view × n_channels) per slice — not optimized; inherent pairwise structure.

---

## Projection2D (`src/Projection2D.cxx`)

### Purpose

Build 2D sparse matrix projections of 3D blob clusters onto wire planes.
Used as input to `ProjectionDeghosting`.

### `get_projection()`

For each (slice, blob, channel):
1. Get channel charge from slice activity
2. If uncertainty > `uncer_cut` (1e11) → mark as dead (`dead_default_charge = -1e12`)
3. Accumulate charge per plane as Eigen sparse triplets `{channel_index, time_slice_index, charge}`
4. Also compute: estimated minimum/total charge, blob/slice counts

Output: one `sparse_mat_t` (Eigen CSC sparse matrix) per wire plane.

### `get_geom_clusters()`

Groups blobs by connectivity via `boost::connected_components` on blob-only subgraph.

### Coverage Judgment

**`judge_coverage(ref, tar)`** — binary mask comparison:
1. Create binary masks (pixel live if charge > `-uncer_cut`)
2. Compare `ref_mask - tar_mask`; threshold 0.01 (hardcoded)
3. Returns: REF_COVERS_TAR / TAR_COVERS_REF / REF_EQ_TAR / BOTH_EMPTY / OTHER

**`judge_coverage_alt(ref, tar)`** — charge and count ratios (more nuanced):
1. Count live/dead pixels, intersection pixels
2. Fractional comparison:
   ```
   (1 - common_charge/small_charge) < min(cut[0]×(small+dead)/small, cut[1])
   AND
   (1 - common_counts/small_counts) < min(cut[2]×(small+dead)/small, cut[3])
   ```

> **Inconsistency**: `judge_coverage` counts zero-charge pixels as "live"; `judge_coverage_alt` only counts `x > 0`. Likely intentional (alt is stricter) but undocumented.

---

## ProjectionDeghosting (`src/ProjectionDeghosting.cxx`)

### Purpose

Global deghosting: compare 2D projections across all cluster pairs and remove
clusters whose projections are subsets of higher-quality ones.

### Method

1. Get geometric clusters (grouped by blob connectivity)
2. Build per-cluster 2D projections on all 3 planes
3. For each plane, compare projection pairs with `judge_coverage()` or `judge_coverage_alt()`
4. If cluster A's projection is covered by cluster B's on enough planes → mark A for removal
5. Apply global chi-squared-like cuts (charge per blob, time slice count)

### Complexity

O(n_clusters²) per plane — inherent to pairwise comparison approach.

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `uncer_cut` | 1e11 | Dead pixel detection threshold |
| `dead_default_charge` | -1e12 | Charge for dead pixels |
| `judge_alt_cut_values` | 4 values | Tolerance for alt coverage |

---

## ShadowGhosting (`src/ShadowGhosting.cxx`)

**Status: Incomplete / pass-through.**

Creates blob and cluster shadow graphs via `BlobShadow::shadow()` and `ClusterShadow::shadow()`
but does not use them — input cluster is passed through unchanged. Scaffolding for
future implementation only.

> If configured with empty string shadow type, returns early with error log (fixed bug).

---

## Deghosting Strategy Summary

| Component | Scope | Method | Status |
|-----------|-------|--------|--------|
| InSliceDeghosting | Per-slice | Wire-score + adjacency | Active |
| ProjectionDeghosting | Global | 2D projection coverage | Active |
| ShadowGhosting | Global | Cross-view shadows | Not functional |

Typical pipeline: `InSliceDeghosting` first (local cleanup before charge solving),
then `ProjectionDeghosting` after charge solving (global cleanup with better charge estimates).

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[Charge Solving]]
- [[Slicing and Tiling]]
- [[Img Bug List]]

## Sources

- [[source-img-examination]]
