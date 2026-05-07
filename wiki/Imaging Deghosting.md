---
tags: [algorithm]
sources: 2
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
Designed to run at multiple points in the imaging pipeline (controlled by `config_round`).

### Bit-tag System

Each blob carries a packed integer tag (enum `BLOB_QUALITY_BITPOS`):

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | GOOD | Charge above `good_blob_charge_th` |
| 1 | BAD | (reserved) |
| 2 | POTENTIAL_GOOD | Blob or a b-b neighbor has good charge |
| 3 | POTENTIAL_BAD | Low wire score, likely ghost |
| 4 | TO_BE_REMOVED | Marked for deletion this round |

Tag helper `tag(map, vtx, bitpos, value)`:
- Set: `pack |= (1 << p)` — **Bug fixed**: was `pack &= (0 << p)` which always zeroed all bits.
- Clear: `pack &= ~(1 << p)`

### Phase 1: Blob Quality (`blob_quality_ident`)

Runs in all rounds before local deghosting:
- Blob charge > `m_good_blob_charge_th` (default 300 e⁻) → tag GOOD + POTENTIAL_GOOD
- If any b-b neighbor has charge > threshold → tag current blob POTENTIAL_GOOD

### Adjacency test (`adjacent()`)

Used to decide if a 2-view blob is "protected" by nearby 3-view blobs:

Per plane, score:
- Score 2: channel sets overlap (any common channel)
- Score 1: channel sets are adjacent (any pair differing by ±1)
- Score 0: no overlap or adjacency → immediate fail for that plane

Sum across all planes present in both blobs. Adjacent if sum ≥ 5.

### Wire overlap (`calculate_wire_overlap()`)

Merge-sorted intersection count. Returns `|wires1 ∩ wires2| / |wires1|` in O(n+m).
Fixed: was div-by-zero when `wires1` was empty (old code checked `wires1.empty()` only after allocation).

---

### Round 1 (`config_round == 1`): Pre-charge-solving cleanup

Sequence: `blob_quality_ident` → `local_deghosting` → delete TO_BE_REMOVED → grouped recluster

**`local_deghosting` per slice:**

1. Group blobs: `view_groups[3]` (all 3 planes live) and `view_groups[2]` (2 planes live)
2. Build `used_plane_channels`: channels covered by GOOD 3-view blobs
3. For 2-view blobs not POTENTIAL_GOOD:
   - If all their live-plane wires are already in `used_plane_channels` → TO_BE_REMOVED (cannot_remove exemption applies)
4. Build `wire_score_map` (per wire, how many surviving 2-view blobs use it)
5. For surviving 2-view blobs: compute per-plane score = Σ(1/wire_score) / n_wires over unused wires
   - If score ≤ `m_deghost_th1` (0.5) for all planes AND not POTENTIAL_GOOD → POTENTIAL_BAD

**cannot_remove exemption**: a 2-view blob adjacent to ≥2 GOOD 3-view blobs is never removed.

**Grouped recluster**: Blobs are sorted into 4 quality groups (POTENTIAL_GOOD only, POTENTIAL_GOOD+POTENTIAL_BAD, neither, POTENTIAL_BAD only) and new b-b edges added by `grouped_geom_clustering`.

---

### Round 2 (`config_round == 2`): Post-charge-solving cleanup

Sequence: `blob_quality_ident` → `local_deghosting1` → delete TO_BE_REMOVED

**`local_deghosting1` per slice:**

1. Group blobs by view count; build `wire_score_map` from ALL 3-view blobs (not just GOOD)
2. Compute `blob_high_score_map` for each blob: max across planes of (average 1/wire_score)
3. For 2-view blobs: compare against neighbor 3-view blobs that share a measurement node (`meas_t`):
   - If neighbor's `blob_high_score_map > m_deghost_th` (0.75)
   - AND `overlap_ratio ≥ m_deghost_th` (0.75)
   - AND neighbor charge `q2 > q1 × m_deghost_th` (i.e., 3-view blob is heavier)
   - Count such neighbors (breaking on first hit per measurement plane)
   - If count == 2 AND blob not in cannot_remove → TO_BE_REMOVED

**cannot_remove** for round 2: adjacent to ≥2 POTENTIAL_GOOD 3-view blobs.

---

### Round 3 (`config_round == 3`): Final cleanup

Sequence: `blob_quality_ident` → keep only POTENTIAL_GOOD blobs

All non-POTENTIAL_GOOD blobs removed. Hard final cut.

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `good_blob_charge_th` | 300 e⁻ | "Good" blob charge threshold |
| `deghost_th` | 0.75 | Wire overlap + charge ratio threshold (round 2) |
| `deghost_th1` | 0.5 | Wire score threshold for POTENTIAL_BAD tagging (round 1) |
| `clustering_policy` | `"uboone"` | Re-clustering policy after round 1 |
| `config_round` | 1 | Which round logic to apply |

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

### Coverage Judgment

**`judge_coverage(ref, tar)`** — binary mask comparison:
1. Create binary masks (pixel live if charge > `-uncer_cut`)
2. Compare `ref_mask - tar_mask`; threshold 0.01 (hardcoded)
3. Returns: `REF_COVERS_TAR` / `TAR_COVERS_REF` / `REF_EQ_TAR` / `BOTH_EMPTY` / `OTHER`

**`judge_coverage_alt(ref, tar)`** — charge and count ratios (more nuanced):
1. Count live/dead pixels, intersection pixels
2. Fractional comparison:
   ```
   (1 - common_charge/small_charge) < min(cut[0]×(small+dead)/small, cut[1])
   AND
   (1 - common_counts/small_counts) < min(cut[2]×(small+dead)/small, cut[3])
   ```
   Default `judge_alt_cut_values = [0.05, 0.33, 0.15, 0.33]`.

> **Note**: `judge_coverage` counts zero-charge pixels as "live"; `judge_coverage_alt` only counts `x > 0`. Alt is stricter by design.

---

## ProjectionDeghosting (`src/ProjectionDeghosting.cxx`)

### Purpose

Global deghosting: compare 2D projections across all clusters and remove those whose
projections are subsets of higher-quality ones. Uses two comparison passes.

### Method

1. Build `BlobShadow` graph (wire sharing) → `ClusterShadow` (cluster-level shadow graph)
2. Compute 2D projections per cluster per plane (cached in `id2lproj` map)
3. **Primary loop** — for each incoming cluster, compare against all already-accepted clusters per plane:
   - `REF_COVERS_TAR`: new is subset of existing → `flag_save = false` (mark as ghost)
   - `REF_EQ_TAR`: same coverage → mark new as ghost AND add existing to same-as list
   - `TAR_COVERS_REF`: new covers existing → remove existing from map
4. **Secondary loop** — pairwise comparison of surviving clusters with `judge_coverage_alt`:
   - `REF_COVERS_TAR` → mark smaller as ghost
   - `TAR_COVERS_REF` → mark smaller as ghost
5. **Counting pass** — for each cluster, `m_saved_flag` increments for each plane that survived, `m_saved_flag_1` for each plane that lost
6. **Final deletion** — based on `flag_saved - flag_saved_1`:

| Planes survived | Retained if |
|-----------------|-------------|
| 3 (all) | √(n_timeslices/cut[0])² + (min_charge/n_blobs/cut[1])² ≥ 1 AND min_charge/n_blobs ≥ cut[2] |
| 2 | Same, using cut[3..5] |
| 1 | Same, using cut[6..8] |
| 0 | Always ghost |

Default `global_deghosting_cut_values` (3 sets × 3 = 9 values) control these thresholds.

### Complexity

O(n_clusters²) per plane — inherent to pairwise comparison approach. No optimization implemented.

---

## ShadowGhosting (`src/ShadowGhosting.cxx`)

**Status: Incomplete / pass-through.**

Creates blob and cluster shadow graphs via `BlobShadow::shadow()` and `ClusterShadow::shadow()`
but does not use them — input cluster is passed through unchanged (`out = in`).
Scaffolding for future implementation only.

---

## Deghosting Strategy Summary

| Component | When | Scope | Method |
|-----------|------|-------|--------|
| InSliceDeghosting (round 1) | Before charge solving | Per-slice, 2-view blobs | Wire-score + channel coverage |
| InSliceDeghosting (round 2) | After charge solving | Per-slice, 2-view blobs | Overlap + charge ratio |
| InSliceDeghosting (round 3) | Final | Global | Drop all non-POTENTIAL_GOOD |
| ProjectionDeghosting | After charge solving | Global, cluster pairs | 2D projection coverage |
| ShadowGhosting | — | — | Not functional |

Typical pipeline: InSliceDeghosting round 1 → GridTiling → ChargeSolving → InSliceDeghosting round 2/3 → ProjectionDeghosting.

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[Charge Solving]]
- [[Slicing and Tiling]]
- [[Clus Deghosting]]
- [[Deghosting Comparison]]
- [[Img Bug List]]

## Sources

- [[source-img-examination]]
- [[source-session-2026-05-07-deghosting]]
