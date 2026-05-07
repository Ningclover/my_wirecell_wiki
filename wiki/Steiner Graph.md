---
tags: [algorithm]
sources: 1
updated: 2026-05-07
---

# Steiner Graph

The Steiner graph is a minimal spanning skeleton built from a cluster's blobs. It is the primary input for the [[Pattern Recognition PR Loop]] and vertex finding.

## Construction (CreateSteinerGraph component)

**Source:** `clus/src/CreateSteinerGraph.cxx`, `clus/src/SteinerGrapher.cxx`

### Step 1: Retiling via ImproveCluster_2

Before building the graph, the cluster is "retiled" to produce a more uniform point spacing. A `BlobSampler` resamples each blob to ~3 mm target spacing. This repairs gaps introduced by dead wires or imaging artifacts. The result is stored as the `"default"` named point cloud on the cluster.

### Step 2: Voronoi tessellation

`create_enhanced_steiner_graph()` (`SteinerGrapher.cxx:898`) takes the base neighbor graph and the set of **steiner terminal** vertices (branch points and endpoints identified in the base graph). It runs Dijkstra simultaneously from all terminals — producing a Voronoi diagram over the graph, where each vertex is assigned to its nearest terminal (`vor.terminal[v]`) and its distance (`vor.distance[v]`).

### Step 3: Inter-terminal path construction

For each edge in the base graph that crosses a Voronoi boundary (connects vertices owned by different terminals), the total path distance is computed as:
```
total = vor.distance[u] + edge_weight + vor.distance[v]
```
The best (minimum-cost) edge for each terminal pair is selected. Then for each selected edge, the Voronoi back-path to each terminal is traced, collecting all intermediate graph edges.

### Step 4: Build reduced graph

All unique edges from the Voronoi paths (deduped, sorted by vertex-index pair for determinism) are assembled into the reduced Steiner graph with **charge-weighted** edge costs:
```
final_distance = geometric_distance × (factor1 + factor2 × 0.5×(Q0/(Qs+Q0) + Q0/(Qt+Q0)))
```
where `Q0=10000`, `factor1=0.8`, `factor2=0.4` (from WCP prototype).

The result is stored as the `"steiner_graph"` named graph on the cluster (`Cluster::give_graph("steiner_graph", ...)`).

## Vertex degree semantics

Vertex degree in the Steiner graph carries physical meaning:

| Degree | Interpretation |
|--------|----------------|
| 1 | Endpoint (track end, shower tip) |
| 2 | Intermediate point on a straight segment |
| ≥ 3 | Branching point — candidate neutrino vertex or hadronic scatter |

The [[Pattern Recognition PR Loop]] uses degree-≥3 vertices as initial neutrino vertex candidates.

## Why Voronoi + path expansion?

True Steiner tree construction is NP-hard. The Voronoi-based approach approximates it: each terminal "claims" its nearby region, and only inter-region boundary edges contribute to the reduced graph. This gives a skeleton that connects all terminal points with near-minimal total path length while preserving the topology of the underlying particle trajectories. The charge weighting steers the graph away from low-charge (noise) regions toward real signal.

Note: `recover_steiner_graph` (which uses Kruskal MST over the terminal graph) exists in the WCP prototype but is **not ported** to the WireCell Toolkit — it is not used in any current NeutrinoTagger/PR code path.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Pattern Recognition PR Loop]]

## Sources

- [[source-clus-examination]]
- verified 2026-05-07 against `SteinerGrapher.cxx:20-124, 898-1074`, `CreateSteinerGraph.cxx:151-178`, `clus/docs/steiner_graph.md`, `clus/docs/patternrecognition/steiner_graph_review.md`
