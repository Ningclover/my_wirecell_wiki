---
tags: [algorithm]
sources: 2
updated: 2026-05-07
---

# Clus Deghosting

Ghost removal algorithms in the `clus` pattern recognition module.
Operates on 3D blob clusters and PR graph segments — two passes at different pipeline stages.

**Relevant files:** `clus/src/clustering_deghost.cxx`, `clus/src/NeutrinoDeghoster.cxx`

## Background

The `clus/` deghosting algorithms remove ghosts that survived the `img/` deghosting passes.
Where `img/` operates on individual blobs within slices, `clus/` operates at coarser granularity:

| Stage | Granularity | When |
|-------|-------------|------|
| ClusteringDeghost | Whole clusters | After imaging, before pattern recognition |
| deghost_clusters | Cluster segments | Inside PR, after Steiner graph |
| deghost_segments | Terminal segments | Inside PR, after deghost_clusters |

All use the same technique: **global 2D point cloud overlap testing** with per-plane distance thresholds.

---

## ClusteringDeghost (`clustering_deghost.cxx`)

### Place in pipeline

Stage 3 of the 6-stage clustering pipeline — after imaging, before pattern recognition.
Runs as an `IEnsembleVisitor` on the `"live"` grouping.

### Method overview

1. Sort clusters longest-first (`stable_sort` for determinism; tiebreak by insertion index)
2. Build 2 global `DynamicPointCloud` objects from the first (longest) cluster
3. For each subsequent cluster: test its 3D points against global clouds in all 3 planes per-APA/face
4. Classify as ghost/real/merge candidate based on per-plane unique counts
5. Ghost clusters are removed or merged; surviving clusters added to global clouds

### Global cloud construction

- **`global_point_cloud`**: all 3D blob points
- **`global_skeleton_cloud`**: for clusters >30 cm: PCA shortest-path skeleton; for ≤30 cm: all points

### Per-point testing (per plane, `dis_cut = 1.2 cm`)

Two-level lookup for each non-dead point:

| Pass | Cloud | Threshold | Match action |
|------|-------|-----------|--------------|
| 1 | `global_point_cloud` | `dis_cut/3 = 0.4 cm` | Credit matching cluster |
| 2 | `global_skeleton_cloud` | `dis_cut×2.0 = 2.4 cm` | Credit skeleton cluster |
| — | — | > 2.4 cm | `num_unique[plane]++` |

Dead wire check (per APA/face, per wire index) → `num_dead[plane]++` instead.

Early exit when `Σnum_unique ≥ 0.24×npts AND Σnum_unique > 25`.

### Ghosting decision

A cluster fails the primary ghost test if (complex compound condition):

```
All three planes satisfy: unique ≤ 10% live, or ≤ 10% total and ≤ 8 pts
AND total unique ≤ 5% total-live, or < 15% total and ≤ 8
AND Σunique ≤ 500
```

Plus several weaker sub-conditions (one plane zero + combined 12%/24% cuts).

If ghosted:
- **Merge check**: if 2 planes agree ≥80% of points land on same other cluster AND that cluster within 20 cm → add merge edge
- **Otherwise**: mark for `live_grouping.destroy_child()`

If *saved* but short (<25 cm) and low unique fraction (<15%): apply secondary merge check (25% combined-overlap threshold).

### Single-APA restriction

```cpp
if (apas.size() > 1) raise<ValueError>("apas.size() %d > 1", apas.size());
```

Explicitly rejects multi-APA groupings at runtime.

### Determinism fixes

- `std::stable_sort` instead of `std::sort` for equal-length cluster tiebreaking
- `unordered_map<const Cluster*, int>` with insertion-index tiebreaker in `find_max_cluster` lambda
- Replaces former `std::map<const Cluster*, int>` which iterated pointer addresses (non-deterministic)

---

## NeutrinoDeghoster (`NeutrinoDeghoster.cxx`)

Entry point: `PatternAlgorithms::deghosting()` calls `deghost_clusters` then `deghost_segments`.

### Place in pipeline

Inside `TaggerCheckNeutrino`, stage 6 sub-pipeline step 4 — after Steiner graph, before proto-vertex finding.

---

### `deghost_clusters`

Segment-based cluster deghosting in the PR graph. Orders by total segment track length (not raw cluster size).

**3 global DynamicPointCloud objects:**

| Cloud | Contents |
|-------|----------|
| `global_point_cloud` | Raw 3D blob points of all clusters |
| `global_steiner_point_cloud` | Steiner tree node points |
| `global_skeleton_cloud` | Segment `wcpts` (raw path waypoints) |

Clusters not in the ordered list (no segments) are added to `global_point_cloud` first.

**Distance thresholds (`dis_cut = 1.2 cm`):**

| Cloud | Threshold |
|-------|-----------|
| `global_point_cloud` | `dis_cut/2 = 0.6 cm` |
| `global_steiner_point_cloud` | `dis_cut×2/3 = 0.8 cm` |
| `global_skeleton_cloud` | `dis_cut×6/4 = 1.8 cm` |

Tests `wcpts` of each segment (raw, not T0-corrected). Skips points outside detector volume (`apa == -1 || face == -1`).

**Ghosting criteria — any of 4 conditions triggers removal:**

| # | Condition |
|---|-----------|
| 1 | `max_dead ≥ 0.8` AND `max_unique ≤ 0.35` AND `avg_unique ≤ 0.16` AND `min_unique ≤ 0.08` |
| 2 | `max_unique ≤ 0.1` AND `avg_unique ≤ 0.05` AND `min_unique ≤ 0.025` |
| 3 | `0.7 ≤ max_dead < 0.8` AND `max_unique ≤ 0.2` AND `avg_unique ≤ 0.1` AND `min_unique ≤ 0.05` |
| 4 | One plane 100% dead, other two both zero unique, `max_unique < 0.75` |

**Output:** Removes all segments of ghost clusters from the PR graph; orphaned vertices cleaned up via `ordered_nodes(graph)` for consistent ordering.

---

### `deghost_segments`

Fine-grained pruning of terminal arms that likely extend ghost trajectories.

**Candidate criteria (all must be true):**

| Criterion | Threshold |
|-----------|-----------|
| Terminal | `start_n == 1` OR `end_n == 1` |
| Low dQ/dx | `< 1.1 × 43e3/cm` (< 1.1 MIP) |
| Length | `> 3.6 cm` |

**Distance thresholds:** identical to `deghost_clusters` (0.6/0.8/1.8 cm) but combined with `||` (flag_in = true if any cloud matches).

**Removal condition:** `num_unique[0] + num_unique[1] + num_unique[2] == 0`
All points overlap with at least one global cloud — no unique contribution.

**Main vertex protection:** If removing the segment would sever the cluster's only connection to the identified main neutrino vertex, the segment is kept.

---

## Key Numerical Parameters

| Parameter | Value | Source |
|-----------|-------|--------|
| `dis_cut` (both) | 1.2 cm | clustering_deghost.cxx:269, NeutrinoDeghoster.cxx:147 |
| ClusteringDeghost primary threshold | 0.4 cm | `dis_cut/3` |
| ClusteringDeghost skeleton threshold | 2.4 cm | `dis_cut×2` |
| Skeleton min-length for path vs all-points | 30 cm | clustering_deghost.cxx:243 |
| Merge distance cut | 20 cm | L497 |
| deghost_segments dQ/dx MIP value | 43e3 e⁻/cm | NeutrinoDeghoster.cxx:421 |
| deghost_segments length cut | 3.6 cm | L421 |

## See Also

- [[WireCell Clus Pipeline Overview]]
- [[Clustering Algorithms Internals]]
- [[Steiner Graph]]
- [[Pattern Recognition PR Loop]]
- [[Imaging Deghosting]]
- [[Deghosting Comparison]]
- [[Neutrino Vertex Determination]]

## Sources

- [[source-clus-examination]]
- [[source-session-2026-05-07-deghosting]]
