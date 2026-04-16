---
tags: [source]
type: file
source: raw/wire-cell-toolkit/clus/
ingested: 2026-04-14
status: complete
updated: 2026-04-14
---

# Source: clus examination

WireCell pattern recognition (PR) module — converts 3D blob clouds into reconstructed particle trajectories and identifies neutrino interactions.

## Files read

| File | Content |
|------|---------|
| `docs/overview.md` | IEnsembleVisitor pattern, 6-stage pipeline, data model |
| `docs/pipeline_stages.md` | Detailed walkthrough of all 6 pipeline stages |
| `docs/data_structures.md` | Facade layer, PC tree, PR::Graph/Vertex/Segment types |
| `docs/pattern_recognition.md` | find_proto_vertex() 9-step loop, vertex scoring, DL variant |
| `docs/steiner_graph.md` | Retiling + MST construction, degree semantics |
| `docs/track_shower_separation.md` | separate_track_shower(), shower grouping, kinematics, PID taggers |
| `docs/track_fitting.md` | TrackFitting utility, Parameters struct, fitting modes, calorimetry |
| `docs/vertex_determination.md` | Per-cluster + global vertex selection, DL vertex (SCN), long muon tracking |
| `docs/particle_identification.md` | is_shower predicate, segment classification, direction propagation BFS |

## Wiki pages created

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Steiner Graph]]
- [[Pattern Recognition PR Loop]]
- [[Track Shower Separation]]
- [[Neutrino Vertex Determination]]
- [[Track Fitting and Calorimetry]]
- [[Particle Identification]]
- [[Neutrino Taggers]]
