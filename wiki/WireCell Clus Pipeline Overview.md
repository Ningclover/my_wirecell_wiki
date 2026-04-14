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
