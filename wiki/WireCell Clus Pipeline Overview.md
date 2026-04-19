---
tags: [algorithm, component]
sources: 1
updated: 2026-04-16
---

# WireCell Clus Pipeline Overview

The `clus` module converts 3D blob clouds produced by imaging into reconstructed particle trajectories and neutrino interaction vertices. It is organized into two functional sub-sections: **Clustering** (stages 1–3) and **Pattern Recognition** (stages 4–6).

## Design pattern: IEnsembleVisitor

All algorithms implement a single interface:

```cpp
struct IEnsembleVisitor {
    virtual void visit(Facade::Ensemble& ensemble) = 0;
};
```

`MultiAlgBlobClustering` is the top-level WireCell component. It holds an ordered list of `IEnsembleVisitor` instances and drives the pipeline by calling `visit()` on each in sequence.

## Data hierarchy

```
Ensemble
  └── Grouping  (tag: "live" or "dead")
        └── Cluster
              └── Blob
```

- **Ensemble**: the entire reconstructed event
- **Grouping**: a set of clusters sharing a semantic tag — `"live"` (signal) or `"dead"` (masked/dead-wire regions)
- **Cluster**: a connected set of blobs; owns a `PR::Graph` (named graphs: `"relaxed_pid"`, `"steiner"`, `"pr"`)
- **Blob**: a single 3D voxel with wire-plane charge measurements

See [[Clus Data Structures]] for the full type hierarchy.

## 6-stage pipeline

### Clustering (stages 1–3)

| Stage | Class | Purpose |
|-------|-------|---------|
| 1 | `ClusteringTaggerFlagTransfer` | Copy tagger flags from dead-wire clusters to live clusters; perform initial blob-to-blob clustering |
| 2 | `ClusteringRecoveringBundle` | Recover blobs near dead wire regions; merge fragmented clusters across dead channels |
| 3 | `ClusteringSwitchScope` | Apply T0 (drift time) correction; switch coordinate frames |

### Pattern Recognition (stages 4–6)

| Stage | Class | Purpose |
|-------|-------|---------|
| 4 | `CreateSteinerGraph` | Build [[Steiner Graph]] skeleton; main graph-topology pass |
| 5 | `MakeFiducialUtils` | Compute fiducial volume geometry used by downstream taggers |
| 6 | `TaggerCheckNeutrino` | Run all particle ID and neutrino tagging algorithms |

## Stage 6 internal sub-pipeline

Within `TaggerCheckNeutrino`, the following run in order:

1. [[Pattern Recognition PR Loop]] — `find_proto_vertex()` + vertex scoring
2. [[Track Shower Separation]] — topology + trajectory classification
3. [[Neutrino Vertex Determination]] — per-cluster and global selection
4. [[Particle Identification]] — BFS direction propagation, PDG assignment
5. [[Neutrino Taggers]] — NuMu, NuE, Pi0, Cosmic, SSM, SinglePhoton

## `MultiAlgBlobClustering` driver (`MultiAlgBlobClustering.cxx`)

The top-level `ITensorSetFilter` node. Its `operator()` method:

1. **Load**: reads imaging cluster files via `as_pctrees`, builds the `Ensemble` with `"live"` and `"dead"` groupings.
2. **Pipeline loop** (one iteration per `IEnsembleVisitor` in `m_pipeline`):
   - Calls `cmeth.meth->visit(ensemble)`.
   - Calls `perf.dump(cmeth.name, ensemble)` — logs cluster counts/point counts per grouping.
   - Calls `grouping->enumerate_idents(m_clusters_id_order)` for all groupings — assigns stable numeric IDs to clusters.
   - Iterates `m_bee_points_configs`: if the config's `visitor` field matches the current visitor name, calls `fill_bee_points` or `fill_bee_points_from_pr_graph` for that grouping.
   - Iterates `m_bee_pf_configs`: if the config's `visitor` matches and the grouping has a track fitting result, calls `fill_bee_pf_tree`.
3. **Post-pipeline**:
   - Iterates remaining `m_bee_points_configs` (those with no `visitor` field, i.e., not visitor-specific).
   - Calls `perf("dump live clusters to bee")`.
   - Optionally calls `grouping2file` if `m_grouping2file_prefix` is set.
   - For each grouping name, calls `ensemble.remove_child(grouping)` then `as_tensors(*node, outpath)` — serializes the grouping to tensors. The grouping is removed from the ensemble before serialization because `as_tensors` requires a root node.
   - Assembles output as `as_tensorset(outtens, ident)`.

### `EnsembleVisitor` struct (header `MultiAlgBlobClustering.h:246`)

```cpp
struct EnsembleVisitor {
    std::string name;   // type/name string from config
    IEnsembleVisitor::pointer meth;  // shared_ptr to visitor instance
};
std::vector<EnsembleVisitor> m_pipeline;
```

Visitors are resolved at configure time via `Factory::find_tn<IEnsembleVisitor>(tn)`.

---

## Key design decisions

- **Visitor pattern** keeps algorithms decoupled; new passes can be inserted without changing the driver.
- **Named PR::Graph per cluster** allows multiple graph representations (`"steiner"` for skeleton, `"pr"` for full trajectory graph) to coexist.
- **Dead-wire clusters** are propagated alongside live ones so recovery algorithms have access to charge-depleted regions.

## See also

- [[Clus Data Structures]]
- [[Steiner Graph]]
- [[Pattern Recognition PR Loop]]
- [[Track Shower Separation]]
- [[Neutrino Vertex Determination]]
- [[Particle Identification]]
- [[Neutrino Taggers]]
- [[PDVD Imaging Configuration]] — upstream imaging step that produces the blob input

## Sources

- [[source-clus-examination]]
