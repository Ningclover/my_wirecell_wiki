---
tags: [source]
type: conversation
date: 2026-04-14
context: debugging heap corruption in clustering_extend.cxx, investigating Find_Closest_Points internals
files_touched: [src/clus/clustering_regular.cxx, src/clus/clustering_extend.cxx, src/clus/Facade_Cluster.cxx]
updated: 2026-04-14
---

# source-session-2026-04-14-clustering-internals

Debugging session investigating a heap corruption crash in the `clus` clustering code. Led to deep reading of `clustering_extend.cxx` and `Facade_Cluster.cxx` to understand the cluster merging geometry algorithms.

## Confirmed findings

- `Find_Closest_Points` swaps cluster1/cluster2 so cluster1 is always the shorter one — see [[Clustering Algorithms Internals]]
- `get_strategic_points` returns axis extremes + snapped convex hull vertices — see [[Clustering Algorithms Internals]]
- `get_hull` uses QuickHull with `useOriginalIndices=true`; indices refer into original `pc[]` array — see [[Clustering Algorithms Internals]]
- `ClusterCache.excluded_points` is set by `connect_graph` and persists across algorithm steps, affecting geometric queries — see [[Clustering Algorithms Internals]]

## Context

Crash occurred in `clustering_extend.cxx` during ICARUS reconstruction. Investigation traced through `Facade_Cluster::get_hull()` QuickHull index semantics and the convergence loop in `Find_Closest_Points`. Heap corruption root cause was investigated but not conclusively resolved in this session.
