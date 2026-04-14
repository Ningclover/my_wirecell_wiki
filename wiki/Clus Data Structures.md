# Clus Data Structures

The `clus` module layers three abstraction levels on top of WireCell's core blob types: the **Facade layer**, the **PC tree**, and the **PR graph types**.

## Facade layer

Facade classes provide ergonomic wrappers around raw WireCell objects, enabling traversal of the data hierarchy (Ensemble → Grouping → Cluster → Blob) without duplicating data.

Key facade classes:

| Class | Wraps | Key accessors |
|-------|-------|---------------|
| `Facade::Ensemble` | entire event | `groupings()`, `live_grouping()`, `dead_grouping()` |
| `Facade::Grouping` | set of clusters | `clusters()`, `tag()` |
| `Facade::Cluster` | connected blob set | `blobs()`, `graph("steiner")`, `graph("pr")` |
| `Facade::Blob` | single 3D voxel | `charge()`, `wires()`, `center()` |

## PC tree (point-cloud tree)

Each cluster maintains a **PC tree** — a k-d tree over blob centers — used for fast spatial queries (nearest-neighbor search during Steiner graph construction and vertex search).

## PR::Graph

Defined as a Boost `adjacency_list` with custom vertex and edge properties. Three named instances per cluster:

| Name | Purpose |
|------|---------|
| `"relaxed_pid"` | Loosened graph used during early PID passes |
| `"steiner"` | Minimal spanning skeleton (see [[Steiner Graph]]) |
| `"pr"` | Full trajectory graph from the PR loop |

## PR::Vertex

Properties stored per graph vertex:

| Field | Type | Notes |
|-------|------|-------|
| `wcpt` | 3D point | World coordinate of the vertex |
| `fit` | fit state | Fitted position after local refinement |
| `kNeutrinoVertex` | bool flag | Set when selected as neutrino interaction vertex |

## PR::Segment

An edge in PR::Graph representing a particle trajectory segment between two `PR::Vertex` nodes.

| Field | Type | Notes |
|-------|------|-------|
| `waypoints` | `vector<wcpt>` | Intermediate points along the segment |
| `fits` | fit states | Per-waypoint fitted positions |
| `dirsign` | ±1 | Propagation direction relative to neutrino vertex |
| `kShowerTrajectory` | bool | Segment identified as shower by trajectory test (KS-test) |
| `kShowerTopology` | bool | Segment identified as shower by topology test (RMS spread) |

## PR::Shower

Aggregates segments and blobs that collectively form a shower object after [[Track Shower Separation]].

## PR::Fit

Stores fitted parameters at a point: position, direction, dQ/dx, dE/dx after recombination model.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Steiner Graph]]
- [[Pattern Recognition PR Loop]]
- [[Track Shower Separation]]
