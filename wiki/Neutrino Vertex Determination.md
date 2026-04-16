---
tags: [algorithm]
sources: 1
updated: 2026-04-16
---

# Neutrino Vertex Determination

Two levels of vertex selection: **per-cluster** (within a single cluster's PR graph) and **global** (across all clusters in the event).

## Per-cluster vertex: determine_main_vertex()

Runs after the [[Pattern Recognition PR Loop]] produces a set of scored proto-vertices.

### improve_vertex()

Iteratively refines the candidate vertex position:

1. Take the current `kNeutrinoVertex` position
2. Collect all segments whose endpoints are within `vtx_search_radius`
3. Refit the vertex as the weighted centroid of segment endpoint positions (weights = segment charge density)
4. Repeat until position converges (< 0.1 cm shift)

### compare_main_vertices() scoring table

When multiple proto-vertices exist, they are ranked by:

| Criterion | Score contribution |
|-----------|-------------------|
| Segments attached | +N per segment |
| Inward-pointing segments (toward fiducial center) | +2 per segment |
| MIP segment present | +3 |
| Dead-wire recovery cluster nearby | +2 |
| Distance from TPC boundary > 5 cm | +5 (fiducial bonus) |
| Distance from TPC boundary < 2 cm | −10 (boundary penalty) |

The highest total score is selected as the cluster's main vertex.

## Global vertex: determine_overall_main_vertex()

Selects the single best vertex across all clusters in the event.

### Multi-cluster selection

1. Collect the main vertex candidate from every `"live"` cluster
2. Score each candidate using `compare_main_vertices()` extended with inter-cluster criteria:
   - Bonus if other clusters point toward this vertex (converging track topology)
   - Bonus if this vertex is inside the fiducial volume while others are not
3. Select the globally highest-scoring candidate

### calc_conflict_maps()

Computes pairwise "conflict" between vertex candidates:

- Two vertices conflict if accepting both would require a track to pass through both (topologically impossible)
- Conflict scores suppress secondary candidates near the selected vertex

## Deep learning variant (SCN_Vertex)

When enabled, a PyTorch-backed Sparse Convolutional Network (SCN) produces per-voxel vertex probabilities.

- The DL vertex is the voxel with maximum SCN score within the cluster
- Accepted only if within `dl_vtx_cut = 2 cm` of the geometric vertex from `determine_main_vertex()`
- If accepted, the DL vertex position replaces the geometric one

## Long muon tracking

For events with a single long track (candidate νμ CC):

1. Identify the longest track segment attached to the neutrino vertex
2. Run `TrackFitting` in multiple-scattering mode along the full muon length
3. The fitted track direction at the vertex is used to constrain the vertex position (vertex pulled toward the track's extrapolated start)

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Pattern Recognition PR Loop]]
- [[Track Fitting and Calorimetry]]
- [[Particle Identification]]
- [[Neutrino Taggers]]

## Sources

- [[source-clus-examination]]
