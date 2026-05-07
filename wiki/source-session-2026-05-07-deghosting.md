---
tags: [source]
type: conversation
date: 2026-05-07
context: Deep dive into all deghosting algorithms across img/ and clus/ packages; comparative analysis
files_touched:
  - img/src/InSliceDeghosting.cxx
  - img/src/ProjectionDeghosting.cxx
  - img/src/ShadowGhosting.cxx
  - img/inc/WireCellImg/InSliceDeghosting.h
  - img/inc/WireCellImg/ProjectionDeghosting.h
  - clus/src/clustering_deghost.cxx
  - clus/src/NeutrinoDeghoster.cxx
  - clus/docs/clustering/clustering_deghost_review.md
  - clus/docs/patternrecognition/deghosting_kinematics_review.md
updated: 2026-05-07
---

# source-session-2026-05-07-deghosting

Deep investigation of all deghosting algorithms across `img/` and `clus/` packages,
including source-level code reading and cross-package comparison.

## Confirmed findings

- InSliceDeghosting has 3 distinct rounds controlled by `config_round`: round 1 (pre-charge-solving, wire-coverage test), round 2 (post-charge-solving, overlap+charge-ratio test), round 3 (final POTENTIAL_GOOD-only keep) — see [[Imaging Deghosting]]
- `local_deghosting` (round 1) and `local_deghosting1` (round 2) are two distinct functions with different logic
- `calculate_wire_overlap()` uses merge-sorted intersection in O(n+m); protected against empty-set div-by-zero
- `adjacent()` scoring: channel overlap → 2 pts, channel adjacency (±1) → 1 pt; needs ≥5 across planes
- ProjectionDeghosting has two passes: primary (judge_coverage binary mask), secondary (judge_coverage_alt with charge/count ratios); final cut uses hyperbolic distance formula on (n_timeslices, min_charge/n_blobs)
- ShadowGhosting is confirmed pass-through; `out = in` unconditionally
- ClusteringDeghost explicitly rejects multi-APA at runtime (`raise<ValueError>` at L172-174)
- `deghost_clusters` uses 3 clouds (point/steiner/skeleton) with dis_cut/2, 2/3, 6/4 thresholds; 4 ghosting conditions
- `deghost_segments` prunes only terminal+low-dQ/dx+long (>3.6 cm) segments; removal requires ALL unique==0
- Main vertex protection in `deghost_segments`: segment kept if removing it disconnects main neutrino vertex

## Context

User requested comprehensive understanding of all deghosting algorithms in img/ and clus/,
their similarities, differences, and ordering in the reconstruction pipeline.
Session read all deghosting source files directly after reading doc review files.

## Pages created/updated

- [[Imaging Deghosting]] — rewritten with accurate multi-round structure
- [[Clus Deghosting]] — new page covering ClusteringDeghost and NeutrinoDeghoster
- [[Deghosting Comparison]] — new synthesis page
