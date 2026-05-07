---
tags: [synthesis, algorithm]
sources: 3
updated: 2026-05-07
---

# Deghosting Comparison

Side-by-side comparison of the five deghosting components across the `img/` and `clus/` packages.

## Pipeline Position

```
IFrame
  → [img/] Slicing & Tiling → blobs
  → [img/] InSliceDeghosting round 1     ← LOCAL GHOST REMOVAL (wire score)
  → [img/] ChargeSolving
  → [img/] InSliceDeghosting round 2/3   ← LOCAL GHOST REMOVAL (charge ratio)
  → [img/] ProjectionDeghosting          ← GLOBAL GHOST REMOVAL (2D projection)
  → ICluster (imaging output)
      → [clus/] ClusteringDeghost        ← CLUSTER-LEVEL GHOST REMOVAL (2D kd-tree)
      → Steiner graph, pattern recognition
      → [clus/] deghost_clusters         ← SEGMENT-LEVEL GHOST REMOVAL (3 clouds)
      → [clus/] deghost_segments         ← TERMINAL SEGMENT PRUNING
```

---

## Comparison Table

| Dimension | InSliceDeghosting | ProjectionDeghosting | ClusteringDeghost | deghost_clusters | deghost_segments |
|-----------|-------------------|---------------------|-------------------|-----------------|-----------------|
| **Module** | img/ | img/ | clus/ | clus/ | clus/ |
| **Scope** | Per slice | Global | Global (single APA) | Global | Per segment |
| **Input granularity** | Individual blobs | Blob clusters | Blob clusters | PR segments | PR terminal segments |
| **Core representation** | Wire channel sets | 2D sparse matrices | 3D point clouds (2D projected) | 3D point clouds (2D projected) | 3D point clouds (2D projected) |
| **Distance measure** | Channel overlap + adjacency score | Binary mask coverage or charge/count ratios | 2D kd-tree nearest neighbor | 2D kd-tree nearest neighbor | 2D kd-tree nearest neighbor |
| **Reference set** | Other blobs in same slice | Previously accepted clusters (ordered) | Longest-first accepted clusters | Longest-segment-first accepted | All accepted clusters |
| **Multi-APA** | Yes (per-slice, APA-agnostic) | Yes | **No** (runtime error if >1 APA) | Yes (per APA/face routing) | Yes |
| **After charge solving?** | Round 1: No; Rounds 2–3: Yes | Yes | No | Yes (PR uses solved charges) | Yes |
| **Output** | Modified cluster graph (blobs deleted, reclustered) | Modified cluster graph (blobs deleted) | Modified grouping (clusters deleted or merged) | Modified PR graph (segments removed) | Modified PR graph (segments removed) |

---

## Ghost Detection Philosophy

### img/ approach: wire coverage

`InSliceDeghosting` exploits that ghosts are 2-view blobs whose wires are already explained by higher-quality 3-view blobs in the same time slice. The "wire score" (1/sharing_count) captures wire uniqueness: a wire appearing in many blobs has low score, flagging the 2-view blob as redundant.

`ProjectionDeghosting` extends this globally: if a cluster's 2D projection onto any wire plane is entirely covered by another cluster's projection, it contributes no unique information and is a ghost.

### clus/ approach: 2D uniqueness counting

Both `ClusteringDeghost` and `deghost_clusters` use the same principle: ghost clusters have points that, when projected into each wire plane, land within distance threshold of points already in the global cloud from real clusters. If a cluster has no "unique" 2D points (below threshold) in any plane, it is a ghost.

The clus/ approach tests 3D Euclidean-derived 2D projections via kd-tree; img/ uses exact wire channel sets (integer channel IDs). The clus/ distance thresholds (0.4–2.4 cm) are more generous than img/ channel-adjacency (1 channel = wire pitch ~ 5 mm).

---

## Key Algorithmic Differences

### Reference set ordering

| Component | Ordering | Rationale |
|-----------|----------|-----------|
| InSliceDeghosting | None (per-slice, simultaneous) | All blobs in slice are contemporaneous |
| ProjectionDeghosting | Iteration order of vertex descriptor | No explicit quality ordering |
| ClusteringDeghost | Longest cluster first | Long clusters more likely real |
| deghost_clusters | Longest total-segment-length first | Segment length is quality proxy post-PR |
| deghost_segments | Longest segments first | Long segments more likely real |

### Granularity progression

Each layer catches ghosts the previous one missed:

- **InSliceDeghosting**: removes 2-view wire ambiguities *within* a single time slice
- **ProjectionDeghosting**: removes clusters with no unique 2D projection globally
- **ClusteringDeghost**: after imaging is done, uses 3D cluster structure to find duplicates
- **deghost_clusters**: after pattern recognition provides the "skeleton" of real tracks, uses it to catch clusters that overlap skeletons
- **deghost_segments**: fine-grained, prunes the low-charge terminal arms of tracks that extend into ghost regions

### Distance thresholds comparison

| Component | Tight threshold | Loose threshold |
|-----------|----------------|----------------|
| InSliceDeghosting | Channel adjacency (≤1 ch) | Channel overlap |
| ProjectionDeghosting | Pixel coverage (binary) | Charge/count ratio |
| ClusteringDeghost | 0.4 cm (1/3 dis_cut) | 2.4 cm (2× dis_cut) |
| deghost_clusters | 0.6 cm (1/2 dis_cut) | 1.8 cm (6/4 dis_cut) |
| deghost_segments | Combined OR of 0.6/0.8/1.8 cm | — |

### Multi-cloud architecture (clus/ only)

`deghost_clusters` and `deghost_segments` use 2–3 global clouds with different geometric resolutions:

| Cloud | Geometry | Tightest threshold |
|-------|----------|--------------------|
| `global_point_cloud` | All 3D blob points | 0.4–0.6 cm |
| `global_steiner_point_cloud` | Steiner tree topology | 0.8 cm |
| `global_skeleton_cloud` | Segment path waypoints | 1.8–2.4 cm |

Hierarchical: fail fast against the tight cloud, fall back to skeleton for distant but correlated points.

---

## Ghosting Criteria Stringency

From loosest to strictest (approximately):

1. **InSliceDeghosting round 3**: any blob without POTENTIAL_GOOD → removed (hardest cut)
2. **ClusteringDeghost**: multi-condition compound gate with unique fraction thresholds (10%/5%/15% total)
3. **deghost_clusters**: 4 ghosting criteria; most aggressive for high-dead-fraction events
4. **deghost_segments**: any terminal arm with zero unique points → pruned
5. **InSliceDeghosting rounds 1–2**: conservative (protected blobs, adjacency exemptions)
6. **ProjectionDeghosting**: cluster must be *fully covered* in 2D projection to be ghosted

---

## Known Limitations

| Limitation | Affects |
|-----------|---------|
| O(n²) pairwise projection comparison | ProjectionDeghosting |
| Single-APA restriction | ClusteringDeghost |
| No deterministic ordering in projection primary loop | ProjectionDeghosting (covered in doc only, not yet fixed) |
| `ShadowGhosting` is a stub | ShadowGhosting |
| `cal_corr_factor()` returns 1.0 (U-plane gain correction) | deghost_clusters/segments |

## See Also

- [[Imaging Deghosting]]
- [[Clus Deghosting]]
- [[WireCell Imaging Pipeline Overview]]
- [[WireCell Clus Pipeline Overview]]
- [[Img Efficiency Issues]]

## Sources

- [[source-img-examination]]
- [[source-clus-examination]]
- [[source-session-2026-05-07-deghosting]]
