# Wiki Log

Append-only chronological record of all wiki operations.

Format: `## [YYYY-MM-DD] <operation> | <description>`

Quick grep for recent entries: `grep "^## \[" wiki/log.md | tail -10`

---

## [2026-04-14] init | WireCell wiki scaffolded
Initial directory structure, CLAUDE.md schema (WireCell domain), index.md, and log.md created.

## [2026-04-14] ingest | ProtoDUNE-VD WireCell configuration files
Ingested `raw/dunereco/dunereco/DUNEWireCell/protodunevd/` — all Jsonnet configs and data files.

Pages created:
- source-pdvd-wct-config.md
- pdvd-wct-configuration-overview.md
- pdvd-detector-params.md
- pdvd-nf-configuration.md
- pdvd-sp-configuration.md
- pdvd-sp-filters.md
- pdvd-dnnroi.md
- pdvd-imaging-configuration.md

## [2026-04-14] ingest | WireCellSigProc code examination
Ingested `raw/wire-cell-toolkit/sigproc/` — examination docs + headers + source overview.

Pages created:
- source-sigproc-examination.md
- wirecellsigproc-pipeline-overview.md
- omnibus-sigproc.md
- roi-formation.md
- roi-refinement.md
- l1sp-filter.md
- omnibus-noise-filter.md
- omnichannel-noisedb.md
- detector-specific-sigproc.md
- sigproc-bug-list.md
- sigproc-efficiency-issues.md

## [2026-04-14] ingest | WireCell clus pattern recognition module
Ingested `raw/wire-cell-toolkit/clus/docs/` — all 9 documentation files covering the full pattern recognition pipeline.

Pages created:
- source-clus-examination.md
- WireCell Clus Pipeline Overview.md
- Clus Data Structures.md
- Steiner Graph.md
- Pattern Recognition PR Loop.md
- Track Shower Separation.md
- Neutrino Vertex Determination.md
- Track Fitting and Calorimetry.md
- Particle Identification.md
- Neutrino Taggers.md

## [2026-04-14] reorganize | Module-based index structure
Restructured index.md and CLAUDE.md to reflect module hierarchy.

- Algorithms section split by module (sigproc, clus) with sub-sections per module
- sigproc: "Noise Filtering" + "Signal Processing" sub-sections
- clus: "Clustering" + "Pattern Recognition" sub-sections
- Placeholder entries for imaging and charge-light matching modules
- New "Experiments & Configuration" section replaces "ProtoDUNE-VD Configuration"
  (groups detector params + pipeline configs per experiment; PDVD now, pdhd pending)
- Renamed "WireCell Pattern Recognition Overview.md" → "WireCell Clus Pipeline Overview.md"
  (page covers all 6 clus stages, not just pattern recognition)
- Updated all 9 pages that linked to the old name

## [2026-04-14] ingest | Clustering Algorithms Internals
- Created [[Clustering Algorithms Internals]] from live debugging session
- Covers: clustering_regular/Find_Closest_Points/get_strategic_points/get_hull internals, cluster swap logic, ClusterCache/excluded_points, crash investigation log (heap corruption, 2026-04-14)
- Updated index.md Clustering sub-section

## [2026-04-15] ingest | Clustering driver and PDVD clustering pipeline
- Updated [[Clustering Algorithms Internals]]: added merge_clusters algorithm (ClusteringFuncs.cxx:48), ClusteringPointed pruning pass, Cluster::wire_plane_id lazy cache, points_property<int>("wpid") → flat_vector → check_size chain, wpid storage type (int/4 bytes)
- Updated [[WireCell Clus Pipeline Overview]]: added MultiAlgBlobClustering driver structure (pipeline loop, post-pipeline steps, EnsembleVisitor struct)
- Created [[PDVD Clustering Configuration]]: per-APA pipeline order (Pointed → LiveDead → Extend → Regular×2), cluster input format (JSON), commented-out visitors
- Updated index.md (new PDVD page, date bump)

## [2026-04-16] relink | Cross-server re-link pass + session record convention
- Added YAML frontmatter to all 9 clus pages (was missing)
- Added `## Sources` → `[[source-clus-examination]]` to all 9 clus pages
- Added `type: file` to source-sigproc-examination, source-pdvd-wct-config, source-clus-examination
- Created [[source-session-2026-04-14-clustering-internals]] — first session record, retroactively documents the 2026-04-14 clustering debugging session
- Updated [[Clustering Algorithms Internals]] Sources to reference session record
- Added [[PDVD Imaging Configuration]] → [[WireCell Clus Pipeline Overview]] cross-link (imaging feeds clus)
- Added [[WireCell Clus Pipeline Overview]] → [[PDVD Imaging Configuration]] in See also
- Fixed broken wikilinks: removed [[SignalROI]], [[PeakFinding]], [[PDVD chndb-base]] (no pages exist); replaced [[MicroBooNE]]/[[ProtoDUNE-SP]]/[[ProtoDUNE-HD]]/[[ProtoDUNE-VD]] in sigproc pages with plain text or redirect to [[ProtoDUNE-VD WireCell Configuration Overview]]
- Normalized See also sections to bullet-list format on OmnibusSigProc, ROI Formation, ROI Refinement, PDVD Imaging, Detector-Specific Signal Processing
- Updated CLAUDE.md: added session record format, Re-link operation, updated Sources index description
- Updated add_to_wiki/SKILL.md: added Step 7 (create session record) and Step 8 (append log)

## [2026-04-17] update | PDVD clustering config entry point and pipeline isolation technique
- Updated [[PDVD Clustering Configuration]]: added Entry point section (wct-clustering.jsonnet → clus.jsonnet, per-face and all-APA MABC instances, cm_pipeline → m_pipeline mapping); added Pipeline isolation technique section (comment-from-end method, deferred heap crash pattern)

## [2026-04-19] update | D3Vector::operator< and get_strategic_points deduplication
- Updated [[Clustering Algorithms Internals]]: added `D3Vector::operator<` section documenting the correct lexicographic strict-weak-ordering implementation and the requirement that all three `else if` guards must be present
- Updated `get_strategic_points` section: noted that deduplication sort uses `D3Vector::operator<` as comparator
