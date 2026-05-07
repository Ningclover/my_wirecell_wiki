---
tags: [algorithm, component]
sources: 1
updated: 2026-05-07
---

# WireCell Imaging Pipeline Overview

The `img` package converts 2D wire readout frames into 3D spatial objects (blobs)
and clusters, then solves for charge and removes ghost artifacts.

## Purpose

Transform signal-processed frames (output of `sigproc`) into a 3D reconstruction
suitable for particle tracking and calorimetry.

## High-Level Data Flow

```
IFrame (from sigproc)
  → Slicing (MaskSlice / SumSlice)
      → ISlice per tick_span window
  → Tiling (GridTiling)
      → IBlob per 3D wire intersection region
  → Blob Operations (BlobGrouping, BlobClustering)
      → Cluster graph with blob, slice, measure, wire, channel nodes
  → Charge Solving (ChargeSolving / CSGraph)
      → Blob charges via LASSO regression
  → Deghosting (InSliceDeghosting, ProjectionDeghosting)
      → Spurious blobs removed
  → ICluster output
```

## Data Interfaces (Zoo)

| Interface | Description |
|-----------|-------------|
| `ISlice` | Channel activity map `IChannel→(charge, uncertainty)` over one tick_span window |
| `IStrip` | Subset of channels from one wire plane with values; intermediate in tiling |
| `IBlob` | 3D volume bounded by time slice + per-plane pitch intervals (ray pairs) |
| `IBlobSet` | Collection of `IBlob` with ident number |
| `ICluster` | Cluster graph (see below) |

### Cluster Graph (`cluster_graph_t`)

Boost `adjacency_list` with 5 node types (variant):

| Node type | Tag | Holds |
|-----------|-----|-------|
| blob | `'b'` | `IBlob` pointer |
| slice | `'s'` | `ISlice` pointer |
| measure | `'m'` | `IMeasure` pointer (per-plane grouped signal) |
| wire | `'w'` | `IWire` pointer |
| channel | `'c'` | `IChannel` pointer |

Edge semantics: b-s (blob in slice), b-m (blob contributes to measure),
b-w (blob covers wire), w-c (wire–channel), b-b (geometric overlap across slices).

## Component Index

### Slicers
- `MaskSlice` — adaptive thresholding, active/dummy/masked planes, Wiener-signal detection
- `SumSlice` — simple accumulation, no thresholding

### Tiling & Blob Formation
- `GridTiling` — RayGrid-based blob finding; one IBlobSet per ISlice
- `NaiveStriper` — strip formation from connected wires (graph-based)
- `BlobGrouping` — adds measure nodes (connected_components per plane per slice)
- `BlobClustering` — buffers per frame, calls `geom_clustering()` for b-b edges
- `GeomClusteringUtil` — RayGrid overlap-based blob-blob association with policies

### Charge Solving
- `ChargeSolving` — orchestrates unpack→weight→solve→prune→repack
- `CSGraph` — LASSO solver with Cholesky whitening (Eigen + Ress)
- `BlobSolving` — simpler standalone LASSO variant
- `ChargeErrorFrameEstimator` — per-channel uncertainty from ROI length + fudge factors

### Deghosting
- `InSliceDeghosting` — local wire-score + adjacency ghost removal per slice
- `Projection2D` — 2D sparse matrix projections of 3D clusters onto wire planes
- `ProjectionDeghosting` — global pairwise projection-coverage ghost removal
- `ShadowGhosting` — incomplete/pass-through (scaffolding only)

### Utilities & Sinks
- `BlobDepoFill` — assigns true charge to blobs from simulation depos (s-d-w graph)
- `FrameQualityTagging` — detects noisy periods (quality flags 0–3)
- `FrameMasking` — zeros trace samples within masked time ranges
- `CMMModifier` — modifies channel mask maps (shorted/veto/dead channels)
- `BlobSetReframer`, `BlobReframer`, `BlobSetSync`, `BlobSetMerge`, `BlobSetFanout` — stream management
- `GlobalGeomClustering`, `LocalGeomClustering`, `ClusterFanout`, `ClusterScopeFilter` — re-clustering
- `LCBlobRemoval`, `BlobDeclustering` — downstream cluster manipulation
- `DeadLiveMerging` — dummy implementation (FIXME noted in code)

## Live vs Dead Slice Categories

`MaskSlice` categorizes slices into:

| Category | Planes | Used by GridTiling |
|----------|--------|-------------------|
| 3-view live | all 3 active | full 3-view blob finding |
| 2-view live | 2 active, 1 masked | 2-view tiling |
| 2-view dead | 0 active, 2 masked, 1 dummy | uses dummy_error=1e12 |

Dummy planes: all channels present with activity `0 ± 1e12` so the plane does
not constrain tiling but still propagates through the solving chain.

## RayGrid Framework

Tiling exploits uniform wire geometry. Abstract "rays" replace physical wires.
Two parallel rays define a ROCS (Ray-Orthogonal Coordinate System).
Crossing points of two ROCS form a regular non-orthogonal grid; pitch indices
computed as: `P^{lmn}_{ij} = j·a^{lmn} + i·a^{mln} + b^{lmn}`.
This reduces blob-finding from naïve O(N^{2+3n}) to near-constant time.

## Wrapped Wires

DUNE/ProtoDUNE APAs have wires wrapping around two faces. The `C` association
matrix in solving mixes the two-face blocks when wires wrap. Each face's blob
set is solved semi-independently (block-diagonal `G`), but `C` couples them.

## Known Limitations

- 3-plane assumption is hardcoded in many files (see [[Img Bug List]])
- UBooNE-tuned defaults in `MaskSlice`, `FrameQualityTagging` (see [[Img Efficiency Issues]])
- `ShadowGhosting` is incomplete
- `DeadLiveMerging` is a dummy implementation

## See Also

- [[WireCellSigProc Pipeline Overview]] — upstream module producing the IFrame input
- [[Slicing and Tiling]]
- [[Charge Solving]]
- [[Imaging Deghosting]]
- [[WireCell Clus Pipeline Overview]] — downstream consumer of ICluster output
- [[PDVD Imaging Configuration]]
- [[Img Bug List]]
- [[Img Efficiency Issues]]

## Sources

- [[source-img-examination]]
