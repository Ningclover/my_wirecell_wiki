---
tags: [source]
type: file
updated: 2026-05-07
---

# source-img-examination

Systematic code examination of `wirecell-working/toolkit/img/` (~38 source files, ~16k+ lines).
Read all `docs/examinations/` files, `docs/imaging-overview.org`, `docs/BlobDepoFill.org`, and `README.org`.

## Pages created

- [[WireCell Imaging Pipeline Overview]] — data flow, component zoo, cluster graph structure
- [[Slicing and Tiling]] — MaskSlice, SumSlice, GridTiling, BlobGrouping, BlobClustering, GeomClusteringUtil
- [[Charge Solving]] — ChargeSolving, CSGraph LASSO, BlobSolving, ChargeErrorFrameEstimator
- [[Imaging Deghosting]] — InSliceDeghosting, Projection2D, ProjectionDeghosting, ShadowGhosting
- [[Img Bug List]] — 18 bugs catalogued from examinations (13 fixed, 5 not fixed)
- [[Img Efficiency Issues]] — 15 efficiency issues (7 fixed, 8 not fixed)

## Context

The `img` package implements Wire-Cell's 3D imaging reconstruction:
frame → slices → blobs (tiling) → clusters → charge solving → deghosting → output.
The examinations directory was produced by a prior systematic code review pass.
Several bugs and efficiency issues were already fixed in that pass; the unfixed ones
are pre-existing limitations or require fundamental redesign.
