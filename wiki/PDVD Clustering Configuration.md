---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-17
---

# PDVD Clustering Configuration

The per-APA clustering pipeline for ProtoDUNE-VD, configured in `wcp-porting-img/pdvd/clus.jsonnet`.

## Entry point

`wcp-porting-img/pdvd/wct-clustering.jsonnet` is the top-level config passed to `wire-cell`. It imports `clus.jsonnet` (not `clus-new.jsonnet`) and instantiates:
- One `clus_maker.per_apa(anode)` graph per anode — each containing two `clus_per_face` sub-graphs (face 0 and face 1), each with its own `MultiAlgBlobClustering` instance named `{anode.name}-{face}`.
- One `clus_maker.all_apa(anodes)` graph — a single `MultiAlgBlobClustering` named `clus_all_apa` that receives merged point trees from all anodes.

The per-face MABC `cm_pipeline` in `clus.jsonnet` maps directly to `m_pipeline` (`std::vector<ClusteringMethod>`) in `MultiAlgBlobClustering.cxx`. The `IEnsembleVisitor` loop at line 1647 iterates this vector.

## Per-APA pipeline (`cm_pipeline`)

The `MultiAlgBlobClustering` instance for each APA/face runs these visitors in order:

| Order | Visitor class | Config parameters | Purpose |
|-------|--------------|-------------------|---------|
| 1 | `ClusteringPointed` | default | Remove zero-point blobs and empty clusters |
| 2 | `ClusteringLiveDead` | `dead_live_overlap_offset=2` | Merge live clusters near dead-wire clusters |
| 3 | `ClusteringExtend` | `flag=4, length_cut=60cm, num_try=0, length_2_cut=15cm, num_dead_try=1` | Extend clusters into dead regions |
| 4 | `ClusteringRegular` | `name="-one", length_cut=60cm, flag_enable_extend=false` | Regular proximity-based merging (first pass) |
| 5 | `ClusteringRegular` | `name="_two", length_cut=30cm, flag_enable_extend=true` | Regular proximity-based merging (second pass) |

The two `ClusteringRegular` entries are distinguished by their `name` suffix (`-one`, `_two`), which is appended to the type/name string used as the `IEnsembleVisitor` factory key.

## Cluster input format

Imaging produces `clusters-apa-anode{N}-ms-active.tar.gz` and `clusters-apa-anode{N}-ms-masked.tar.gz` files. Each archive contains a single `cluster_{ident}_graph.json` file (not numpy). The JSON format stores nodes (type `c`=cluster, `b`=blob, `m`=measurement, `w`=wire, `s`=slice) and edges.

Wire plane ID ("wpid") in the JSON is a plain integer: the `WirePlaneId::ident()` value.

## Cluster output

- `mabc-{anode}-face{N}.zip` — Bee visualization zip per face.
- `mabc-{anode}.zip` — Combined per-anode zip.
- `mabc-all-apa.zip` — Combined all-APA zip.

## Commented-out visitors

The following visitors are present in `clus.jsonnet` but commented out for PDVD:

- `cm.ctpointcloud()` — between Pointed and LiveDead
- `cm.parallel_prolong(length_cut=35cm)`
- `cm.close(length_cut=1.2cm)`
- `cm.extend_loop(num_try=3)`
- `cm.separate(use_ctpc=true)`
- `cm.connect1()`
- `cm.isolated()`
- `cm.retile(...)`

## Pipeline isolation technique

To locate which `cm_pipeline` step causes a crash, comment out steps from the **end** of the pipeline (not the middle — steps depend on each other's output). Restore one step at a time from the end until the crash reappears. The step that reintroduces the crash is the crashing step.

Note: glibc heap corruption crashes are **deferred** — the corrupting write happens inside one visitor but `abort()` fires at the next `malloc`/`free` in the following visitor. The last successful visitor in the log is therefore not the guilty one; the crash fires one step later.

## See also

- [[PDVD Imaging Configuration]]
- [[Clustering Algorithms Internals]]
- [[WireCell Clus Pipeline Overview]]
- [[ProtoDUNE-VD WireCell Configuration Overview]]
