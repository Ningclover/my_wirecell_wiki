---
tags: [algorithm]
sources: 1
updated: 2026-05-07
---

# Slicing and Tiling

Converts 2D wire frames into 3D blobs via time slicing and ray-grid intersection.
Part of the [[WireCell Imaging Pipeline Overview]].

## MaskSlice (`src/MaskSlice.cxx`)

### Purpose

Full-featured slicer with adaptive thresholding, plane categorization (active/dummy/masked),
and noise-aware activity detection.

### Method

For each `tick_span`-wide time window:
1. Retrieve Wiener-filtered, charge, error, and summary traces from `IFrame`
2. For each channel, check activity threshold:
   - **Wiener path**: channel active if `wiener > nthreshold × default_threshold[plane]`
   - **Neighbor path**: active if Gaussian-smoothed signal > 1/3 of neighboring slice AND neighbor exceeds threshold
3. Active channels contribute charge + error to the slice activity map
4. **Dummy planes**: all channels get `dummy_charge=0, dummy_error=1e12`
5. **Masked planes**: bad channels (from CMM "bad" tag) get `masked_charge=0, masked_error=1e12`

Output: `TracelessFrame` wrapper — minimal `IFrame` with only slice info, no raw traces.

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `tick_span` | 4 | Ticks per slice |
| `nthreshold` | [3.6, 3.6, 3.6] | Per-plane multiplier |
| `default_threshold` | [2351.3, 3346.6, 2271.9] | UBooNE RMS×4 — detector-specific! |
| `dummy_error` | 1e12 | Marks dummy plane channels unreliable |
| `masked_error` | 1e12 | Marks bad channels unreliable |

> **Warning**: `default_threshold` values are MicroBooNE-specific. Must be overridden for other detectors (PDVD, PDHD, etc.).

---

## SumSlice (`src/SumSlice.cxx`)

Simple accumulation slicer — groups non-zero trace samples into `tick_span` bins,
accumulates channel charge directly without thresholding. No plane categories.
Used for simple simulation pipelines.

---

## GridTiling (`src/GridTiling.cxx`)

### Purpose

Convert per-slice channel activity into 3D blobs using the RayGrid framework.

### Method

For each `ISlice`:
1. Iterate over wire plane layers in the anode face
2. Build `Activity` objects per layer: check if each wire's channel has activity above `threshold`
3. If fewer than `nplanes` layers have activity → return early (empty blob set)
4. Call `RayGrid::make_blobs(face, activities)` — finds 3D regions where active wires from all planes intersect
5. Assign unique incrementing blob IDs
6. Package into `SimpleBlobSet`

**RayGrid blob-finding**: treats each wire plane as parallel rays, finds pairwise strip
intersections, then intersects all planes for 3D volumes.

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `threshold` | 0.0 | Minimum channel activity |
| `nudge` | 1e-3 | Floating-point robustness correction |

---

## BlobGrouping (`src/BlobGrouping.cxx`)

### Purpose

Add "measure" nodes to the cluster graph — electrically connected blob-channel groups
per wire plane, needed by [[Charge Solving]].

### Method

Per slice:
1. For each of 3 planes (hard-coded `bcs(3)` — known limitation), build a bipartite subgraph
   with blob and channel vertices, edges from blob-channel connections
2. Run `boost::connected_components` on each per-plane subgraph
3. Each connected component → one `SimpleMeasure` (signal = sum of channel activity values)
4. Add measure nodes and edges (b-m) to cluster graph

> **Bug**: Hard-coded `bcs(3)` — breaks for non-3-plane geometries. Acknowledged with FIXME comment.

---

## BlobClustering (`src/BlobClustering.cxx`)

### Purpose

Form one `ICluster` per frame by geometric clustering of blobs across time slices.

### Method

1. Buffer incoming blob sets until frame ident changes
2. On new frame: sort buffered sets by slice time
3. Build cluster graph (add slice, blob, channel, wire nodes)
4. Call `geom_clustering()` (via `GeomClusteringUtil`) → b-b edges between adjacent slices
5. Output one `ICluster`

---

## GeomClusteringUtil (`src/GeomClusteringUtil.cxx`)

### Purpose

Blob-to-blob association across time slices using RayGrid overlap.

### Method

For each pair of adjacent blob sets (within `max_rel_diff` slices apart):
- Use `TolerantVisitor` wrapping `RayGrid::overlap()` with wire gap tolerance
- Tolerance from `map_gap_tol[rel_diff]`
- Add b-b edges for all overlapping pairs

### Clustering Policies

| Policy | max_rel_diff | gap_tol |
|--------|-------------|---------|
| `simple` | 1 | {1: 0} |
| `uboone` | 2 | {1: 2, 2: 1} |
| `uboone_local` | 2 | {1: 2, 2: 2} |
| `dead_clus` | special | uses `adjacent_dead()` |

`dead_clus` has a hardcoded dead-time offset `4*500*us` — detector-specific, not configurable.
Policy names reflect MicroBooNE origin.

Also provides `grouped_geom_clustering()` for re-clustering on pre-defined blob groups
(used by `GlobalGeomClustering`, `LocalGeomClustering`).

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[OmnibusSigProc]] — produces `wiener`/`gauss` frames consumed by MaskSlice
- [[ROI Formation]] — `summary_wiener` per-wire RMS used as MaskSlice activity threshold
- [[Charge Solving]]
- [[PDVD Imaging Configuration]]
- [[Img Bug List]]

## Sources

- [[source-img-examination]]
