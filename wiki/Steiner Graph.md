---
tags: [algorithm]
sources: 1
updated: 2026-04-16
---

# Steiner Graph

The Steiner graph is a minimal spanning skeleton built from a cluster's blobs. It is the primary input for the [[Pattern Recognition PR Loop]] and vertex finding.

## Construction (Stage 4: CreateSteinerGraph)

### Step 1: Retiling via ImproveCluster_2

Before building the graph, the cluster is "retiled" to produce a more uniform point spacing:

- Blobs are projected onto wire planes and re-segmented at a finer granularity
- The result is a denser, more uniform set of 3D points covering the cluster volume
- This step repairs gaps introduced by dead wires or imaging artifacts

### Step 2: Neighbor search (k-d tree)

A k-d tree is built over the retiled blob centers. For each point, its k nearest neighbors are found (k is configurable, typically 3–5). This produces a candidate edge set.

### Step 3: Kruskal MST

Kruskal's algorithm is run on the candidate edges (weighted by Euclidean distance) to produce the **minimum spanning tree** — the Steiner graph skeleton.

The MST is stored as the `"steiner"` named graph on the cluster (see [[Clus Data Structures]]).

## Vertex degree semantics

Vertex degree in the Steiner graph carries physical meaning:

| Degree | Interpretation |
|--------|----------------|
| 1 | Endpoint (track end, shower tip) |
| 2 | Intermediate point on a straight segment |
| ≥ 3 | Branching point — candidate neutrino vertex or hadronic scatter |

The [[Pattern Recognition PR Loop]] uses degree-≥3 vertices as initial neutrino vertex candidates.

## Why MST?

The MST minimizes total wire length while connecting all points — it produces a topologically correct skeleton without loops, which simplifies downstream graph traversal and vertex finding. Loops appear only if the retiled point cloud has genuine closed-path topology (e.g., a ring-shaped shower), and they are handled separately.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Pattern Recognition PR Loop]]

## Sources

- [[source-clus-examination]]
