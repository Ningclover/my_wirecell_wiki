---
tags: [algorithm]
sources: 1
updated: 2026-04-16
---

# Pattern Recognition PR Loop

The PR loop transforms the [[Steiner Graph]] skeleton into a fully labeled trajectory graph with identified vertices, track/shower segments, and a candidate neutrino vertex.

Entry point: `find_proto_vertex(Cluster&)`.

## 9-step loop

### Step 1: init_first_segment

Pick the longest straight segment in the Steiner graph as the initial "seed" segment. Build the first `PR::Segment` and add it to the `"pr"` graph.

### Step 2: find_cluster_vertices

Walk the Steiner graph looking for degree-≥3 nodes. Each is a **proto-vertex** — a candidate interaction or scatter point. Convert each to a `PR::Vertex` and insert into the `"pr"` graph.

### Step 3: proto_break_tracks

Break long straight segments at any proto-vertex that falls along their path. This ensures every branching point is an explicit graph vertex, not a waypoint buried inside a segment.

### Step 4–7: examine_structure (×4 iterations)

A repair loop run four times:

- Detect inconsistencies between the Steiner graph topology and the current `"pr"` graph
- Merge vertices that are spatially too close (< merge_threshold)
- Split vertices where the local charge density suggests multiple overlapping tracks
- Re-fit segment waypoints using [[Track Fitting and Calorimetry]]

Running four times converges the graph toward a stable topology.

### Step 8: find_other_segments

After the main vertices are fixed, scan for blobs not yet attached to any segment. Build additional `PR::Segment` objects to cover them, connecting to the nearest existing vertex.

### Step 9 (sub-steps): examine_segment → examine_vertices → break_segments → merge_nearby_vertices

Final cleanup:

- **examine_segment**: Re-evaluate each segment's direction and curvature; flag `kShowerTrajectory` or `kShowerTopology` if criteria met (see [[Track Shower Separation]])
- **examine_vertices**: Verify each vertex has consistent incoming/outgoing segment directions
- **break_segments**: Split segments where dQ/dx changes abruptly (potential scatter point)
- **merge_nearby_vertices**: Final merge pass for vertices within 1 cm of each other

## Vertex scoring

After the loop, each proto-vertex is scored to select the best neutrino vertex candidate. Scoring criteria:

| Criterion | Weight | Notes |
|-----------|--------|-------|
| Number of outgoing segments | high | More branches → more likely to be primary vertex |
| Proximity to fiducial boundary | penalty | Vertices near the TPC edge are likely cosmics |
| Presence of low-dQ/dx (MIP) segments | bonus | MIP tracks expected near neutrino vertex |
| Consistency with dead-wire recovery clusters | bonus | Recovered clusters near vertex increase confidence |

The highest-scoring vertex is stored as `kNeutrinoVertex = true` on the `PR::Vertex`.

## Deep learning variant (SCN)

When DL vertex finding is enabled, a Sparse Convolutional Network (SCN) classifies each blob voxel as "vertex-like" or not. The DL score is combined with the geometric score above. The final DL-enhanced vertex must be within `dl_vtx_cut = 2 cm` of the geometric candidate to be accepted.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Steiner Graph]]
- [[Clus Data Structures]]
- [[Track Shower Separation]]
- [[Neutrino Vertex Determination]]

## Sources

- [[source-clus-examination]]
