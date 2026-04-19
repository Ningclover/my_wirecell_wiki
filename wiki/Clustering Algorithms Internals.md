---
tags: [algorithm]
sources: 2
updated: 2026-04-19
---

# Clustering Algorithms Internals

Internal algorithm details for the blob-cluster merging passes in `toolkit/clus/src/`. Covers `clustering_regular`, `clustering_extend`, `connect_graph`, and supporting geometry utilities.

---

## Cluster merging overview

Clustering works by iterating over pairs of existing clusters and deciding whether to merge them. The primary merging criterion is **proximity**: if the closest distance between two clusters is below a threshold, they are merged. The central routine that measures inter-cluster distance is `Find_Closest_Points`.

---

## `clustering_regular` (`clustering_regular.cxx`)

Entry point: `Clustering_1st_round()`.

- Iterates over all pairs of clusters.
- For each pair, checks cluster lengths against `length_cut`.
- Calls `Find_Closest_Points` (line 75) to compute the minimum distance between the two clusters.
- Merges clusters whose closest distance is below the merge threshold.

---

## `Find_Closest_Points` (`clustering_extend.cxx:902`)

Computes the minimum distance between two clusters and the pair of points achieving it.

### Cluster swap (line ~916)

```cpp
if (length_1 >= length_2) {
    swapped = true;
    std::swap(cluster1, cluster2);
    std::swap(length_1, length_2);
}
```

After the swap, **cluster1 is always the shorter cluster** and **cluster2 is always the longer one**. This is a performance optimization: `get_strategic_points` is called on cluster1, which has fewer hull vertices → fewer starting points → fewer convergence iterations. The KD-tree query is on cluster2 (larger), but lookup cost is `O(log N)` regardless.

### Algorithm

1. Call `get_strategic_points(*cluster1)` → list of `(geo_point_t, const Blob*)` pairs as starting points.
2. For each starting point `(start_p1, start_mcell1)`:
   - Run a **convergence loop** (max 100 iterations):
     - `p2, mcell2 = cluster2->get_closest_point_blob(p1)` — nearest point on cluster2 to current `p1`
     - `p1, mcell1 = cluster1->get_closest_point_blob(p2)` — nearest point on cluster1 to new `p2`
     - Stop when both `mcell1` and `mcell2` stop changing between iterations (blob assignments stabilized).
   - Record `dis = |p1 - p2|` if it is the best seen so far.
3. Return the minimum distance found across all starting points.

### Why convergence works

Each step moves `p1` and `p2` strictly no farther apart, so blob assignments stabilize quickly (typically 2–5 iterations). The result is the local minimum of inter-cluster distance reachable from each strategic starting point.

### `start_mcell1` / `start_mcell2`

Each strategic point is paired with the **blob** that owns it. The blob pointer tracks which cell the closest point belongs to, enabling the convergence check (blob assignments stop changing = converged).

---

## `get_strategic_points` (`clustering_extend.cxx:860`)

Returns a deduplicated list of `(geo_point_t, const Blob*)` pairs representing the "interesting" boundary points of a cluster to use as starting points for `Find_Closest_Points`.

### Sources of strategic points

1. **Axis extremes** (6 points):
   - Highest/lowest Y (`get_highest_lowest_points(1)`)
   - Front/back Z (`get_front_back_points()`)
   - Earliest/latest X (`get_earliest_latest_points()`)
2. **Convex hull vertices** (`get_hull()` → for each hull point, snap to nearest actual cluster point via `get_closest_point_blob`):
   ```cpp
   auto hull_points = cluster.get_hull();
   for (const auto& p : hull_points) {
       auto [closest_p, blob] = cluster.get_closest_point_blob(p);
       points.emplace_back(closest_p, blob);
   }
   ```

After collecting all candidates, the list is **sorted and deduplicated** by `geo_point_t` position using `D3Vector::operator<` as the comparator.

---

## `get_hull` (`Facade_Cluster.cxx:2227`)

Computes and caches the convex hull of a cluster's 3D point cloud.

### Steps

1. **Cache check** — if `hull_points` already populated, return immediately.
2. **Size guard** — if `npoints() > MaxHullPoints`, skip hull computation and return empty cache.
3. **Float copy** — copies the cluster's double-precision `points[0/1/2][i]` arrays into `std::vector<Vector3<float>>` called `pc`. QuickHull only works in float precision.
4. **QuickHull** — calls `qh.getConvexHull(pc, false, true)`:
   - `CCW=false` → clockwise winding
   - `useOriginalIndices=true` → index buffer references indices into the original `pc[]` array (not an internal deduplicated buffer)
5. **Collect indices** — iterates the index buffer (triangle face indices), deduplicates into `std::set<int>`.
6. **Store hull points** — for each unique index `i`, stores `{points[0][i], points[1][i], points[2][i]}` (double precision) into the cache.
7. **Return** cached hull points.

### QuickHull index semantics

With `useOriginalIndices=true`, the `ConvexHull` constructor (in `ConvexHull.h:85`) pushes face vertex indices directly into `m_indices` **without remapping** through the `vertexIndexMapping`. This means indices in `m_indices` are valid indices into the original `pc[]` array — and by construction into `points[0/1/2][]` as well, since `pc[i]` was built from `points[*][i]`.

---

## `merge_clusters` (`ClusteringFuncs.cxx:48`)

Merges clusters connected in a `cluster_connectivity_graph_t` (Boost graph). Called by most clustering visitors after building the connectivity graph.

### Signature

```cpp
std::vector<Cluster*> merge_clusters(
    cluster_connectivity_graph_t& g,
    Grouping& grouping,
    const std::string& aname = "",
    const std::string& pcname = "");
```

### Algorithm

1. Run `boost::connected_components(g, ...)` to find which clusters should be merged.
2. For each connected component with ≥ 2 nodes:
   - **Copy** (not reference) `grouping.children()` into `orig_clusters` — this preserves original order even as children are removed during the loop. The comment in the source explicitly flags this: *"A missing '&' is intentional"*.
   - Create a new `fresh_cluster` via `grouping.make_child()`.
   - For each cluster in the component:
     - Call `fresh_cluster.from(*live)` — copies scope/flags/scalar metadata.
     - Call `fresh_cluster.take_children(*live, true)` — moves all blobs from `live` into `fresh_cluster`, emitting `Action::inserted` notifications.
     - Call `grouping.destroy_child(live)` — removes the now-empty cluster and sets pointer to `nullptr`.
3. Return vector of newly-created `fresh_cluster` pointers.

### Key invariant

After `destroy_child(live)`, the `live` pointer in `orig_clusters[idx]` is **not** set to null (it is a copy, not the same pointer variable). The code relies on never revisiting the same index — it is safe because the loop only visits each index once.

### `take_children` / `adopt_children` / `insert` chain

`take_children(other)` → `remove_children(other)` → `adopt_children(kids)` → `insert(node, notify_value=true)` → emits `NaryTree::Action::inserted` notification on the cluster value. This notification invalidates any cached point cloud data on the cluster (k-d trees, hull, PCA), forcing lazy rebuild on next access.

---

## `ClusteringPointed` (`clustering_pointed.cxx`)

A simple cleanup visitor that removes empty blobs and clusters before the main clustering passes.

### Algorithm

For each grouping name in its configured list (default: `"live"`):

1. Iterate clusters. For each cluster, collect blobs with `blob->npoints() == 0` as "doomed".
2. Destroy all doomed blobs via `cluster->destroy_child(blob)`.
3. Collect clusters with `cluster->nchildren() == 0` as "doomed".
4. Destroy all doomed clusters via `grouping->destroy_child(cluster)`.

No merging or graph construction — pure pruning pass.

---

## `Cluster::wire_plane_id` and `points_property<int>("wpid")`

### Lazy wpid cache (`Facade_Cluster.cxx:846`)

```cpp
WirePlaneId Cluster::wire_plane_id(size_t point_index) const {
    auto& wpids = cache().point_wpids;
    if (wpids.empty()) {
        wpids = points_property<int>("wpid");  // builds on first call
    }
    return WirePlaneId(wpids[point_index]);
}
```

- `cache().point_wpids` is a `std::vector<int>` stored in `ClusterCache`.
- On first call, `points_property<int>("wpid")` traverses the scope view and aggregates the "wpid" array from every blob's local point cloud.
- The cache is **invalidated** whenever the cluster's children change (e.g., after `take_children`), forcing a rebuild on next access.

### `points_property<T>` → `flat_vector<T>` → `elements<T>` chain

```
Cluster::points_property<int>("wpid")
  → sv().flat_vector<int>("wpid")           // PointTree.h:343
      → for each node in scope:
          aptr = lpc.get("wpid")
          aptr->elements<int>()             // PointCloudArray.h:260
              → check_size<int>()           // validates sizeof(int) == m_ele_size
```

`check_size<int>()` throws `WireCell::ValueError("element size mismatch %d != %d")` if the stored element size does not match `sizeof(int) = 4`. This means **every blob** in the cluster must have its "wpid" array stored as `int` (not `double`, `int64_t`, etc.).

### "wpid" storage type

The "wpid" array is stored as `int` (4 bytes). Confirmed path:
- `BlobSampler.cxx:373`: `nd("wpid", wpid_blob.ident())` where `WirePlaneId::ident()` returns `int`.
- `npts_dup::operator()` (template `Num = int`) stores `Array(std::vector<int>(npts, val))`.
- `Facade_Blob.cxx:83`: reads back with `pc_scalar.get("wpid")->elements<int>()[0]`.

---

## `ClusterCache` and `excluded_points`

Each cluster owns a `ClusterCache` (accessed via `cache()`). The cache stores:
- `hull_points` — convex hull vertex positions (lazily computed)
- `pca` — PCA result (lazily computed)
- `excluded_points` — set of point indices to exclude from geometric queries

`excluded_points` is set by `connect_graph` (via `set_excluded_points`) and persists across algorithm steps. It affects `get_highest_lowest_points` and other geometric queries by skipping those point indices.

---

---

## `D3Vector::operator<` (`toolkit/util/inc/WireCellUtil/D3Vector.h:168`)

`geo_point_t` is `WireCell::Point` which is `D3Vector<double>`. Its `operator<` defines the sort order used throughout the clus module whenever `geo_point_t` values are compared or sorted (e.g., in `get_strategic_points` deduplication, `std::sort`, `std::unique`).

### Correct implementation (as of 2026-04-19 fix)

```cpp
bool operator<(const D3Vector& rhs) const
{
    if (x() < rhs.x()) return true;
    else if (x() > rhs.x()) return false;
    else if (y() < rhs.y()) return true;
    else if (y() > rhs.y()) return false;
    else return z() < rhs.z();
}
```

This is a standard lexicographic strict weak ordering on (x, y, z).

### Requirement

Any STL algorithm that uses this comparator (`std::sort`, `std::unique`, `std::lower_bound`, etc.) requires it to satisfy **strict weak ordering**: irreflexivity, asymmetry, and transitivity. Missing the `else if (x() > rhs.x()) return false` guard causes the comparator to fall through to comparing `y` when `x` values are equal — violating asymmetry and producing undefined behavior in `std::sort`.

---

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Steiner Graph]]

## Sources

- [[source-clus-examination]] — algorithm structure from `clus` docs
- [[source-session-2026-04-14-clustering-internals]] — code investigation, heap corruption debugging
