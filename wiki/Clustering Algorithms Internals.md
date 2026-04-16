---
tags: [algorithm]
sources: 2
updated: 2026-04-16
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

After collecting all candidates, the list is **sorted and deduplicated** by `geo_point_t` position.

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

## `ClusterCache` and `excluded_points`

Each cluster owns a `ClusterCache` (accessed via `cache()`). The cache stores:
- `hull_points` — convex hull vertex positions (lazily computed)
- `pca` — PCA result (lazily computed)
- `excluded_points` — set of point indices to exclude from geometric queries

`excluded_points` is set by `connect_graph` (via `set_excluded_points`) and persists across algorithm steps. It affects `get_highest_lowest_points` and other geometric queries by skipping those point indices.

---

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Steiner Graph]]

## Sources

- [[source-clus-examination]] — algorithm structure from `clus` docs
- [[source-session-2026-04-14-clustering-internals]] — code investigation, heap corruption debugging
