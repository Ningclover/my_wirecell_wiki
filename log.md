# Wiki Log

Append-only chronological record of all wiki operations.

Format: `## [YYYY-MM-DD] <operation> | <description>`

Quick grep for recent entries: `grep "^## \[" log.md | tail -10`

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

## [2026-04-25] ingest | PDHD SP deconvolution + ROI pipeline walkthrough
- Updated [[OmnibusSigProc]]: full decon_2D_init step-by-step, m_nticks origin, filter variant table, save_data threshold chain, complete config parameter table
- Updated [[ROI Formation]]: detailed cal_RMS algorithm, RMS propagation chain, stage-by-stage method, loose ROI peak detection details
- Updated [[ROI Refinement]]: r_th_factor dual usage (load_data + CheckROIs), fake_signal_*_ind_factor effect table, ind_factor=0.15 anomaly explanation
- Created [[PDHD Signal Processing Configuration]]: output archive contents, SP filter assignments, key tuning parameters, summary_wiener→imaging threshold chain
- session record: source-session-2026-04-25-pdhd-sp-deconvolution.md

## [2026-04-26] ingest | ADC to electrons signal amplitude chain
- Created: `ADC to Electrons Signal Chain.md` — full unit chain from raw ADC → 2D deconvolution → e⁻/tick
- Updated: `OmnibusSigProc.md` — added unit note after decon_2D_init, cross-link to new page
- Updated: `PDHD Signal Processing Configuration.md` — added signal units section and cross-link
- Updated: `index.md` — new page and session record added
- session record: source-session-2026-04-26-adc-to-electrons.md

## [2026-04-29] ingest | track-sim geometry & pdhd parameters
- Created [[WireCell Wires Schema]] — JSON wires file format, list-position-vs-ident gotcha, per-detector record counts, plane order, HD cathode-facing-face table
- Created [[WireCell Sim Track Conventions]] — `sim.tracks` MIP convention, active-volume constraint (depo-x must be inside `[anode_plane, cathode_plane]`), reframer tick origin, expected hot-channel counts per plane
- Created [[pdhd Detector Parameters]] — full pdhd parameter page: geometry, active drift volume, `elecGain` extVar (must be `-V`/`--ext-str`, not `--ext-code`), `nf.jsonnet:intraces:''` wildcard, channel layout
- Updated [[PDVD Detector Parameters]] — added Active drift volume / x-coordinate layout section with anode/response/cathode plane offsets and reframer tick-origin note
- Added new "General / Cross-cutting" Algorithms sub-section to index.md
- Promoted `ProtoDUNE-HD (pdhd)` index sub-section out of *(pending)* state
- session record: source-session-2026-04-29-track-sim-geometry.md

## [2026-05-03] ingest | PDVD field response computation pipeline (pochoir)
- Created [[PDVD Field Response Computation]] — full 10-stage pipeline doc: domain, gen, FDM, bc-interp, extendwf, velo, drift, induce, convertfr
- Updated [[PDVD Detector Parameters]] — added link to FR computation page in "Field response" section
- Added new PDVD index entry for [[PDVD Field Response Computation]]
- session record: source-session-2026-05-03-pdvd-field-response.md

## [2026-05-07] ingest | WireCell img/ package — 3D imaging pipeline

Source: `wirecell-working/toolkit/img/` (~38 source files, examinations docs)

Pages created:
- [[source-img-examination]] — ingest record
- [[WireCell Imaging Pipeline Overview]] — component zoo, cluster graph, RayGrid, live/dead categories
- [[Slicing and Tiling]] — MaskSlice, SumSlice, GridTiling, BlobGrouping, BlobClustering, GeomClusteringUtil
- [[Charge Solving]] — ChargeSolving, CSGraph LASSO + Cholesky whitening, BlobSolving, ChargeErrorFrameEstimator
- [[Imaging Deghosting]] — InSliceDeghosting, Projection2D, ProjectionDeghosting, ShadowGhosting
- [[Img Bug List]] — 18 bugs (13 fixed, 5 not fixed)
- [[Img Efficiency Issues]] — 15 efficiency issues (7 fixed, 8 not fixed); PDHD porting constants table

Index updated: imaging section filled in (was "pending").

## [2026-05-07] relink

Comprehensive audit of all 50 wiki pages. Issues found and fixed:

**Broken wikilinks (2 files):**
- `[[PMT Noise Filter]]` de-linked in `Omnibus Noise Filter` (content is already in that page) and `WireCellSigProc Pipeline Overview` (redirected to `[[Omnibus Noise Filter]]`)

**Wrong/missing frontmatter (4 files):**
- `SigProc Bug List`, `SigProc Efficiency Issues`: tag `[meta]` → `[synthesis]`
- `PDVD vs PDHD Signal Chain Comparison`: non-schema tag `comparison` → `synthesis`
- `PDVD Field Response Computation`: added missing `sources: 1`

**Module-boundary cross-references added (9 files):**
- `OmnibusSigProc` → `[[WireCell Imaging Pipeline Overview]]`
- `ROI Formation` → `[[Slicing and Tiling]]` (summary_wiener chain)
- `WireCell Imaging Pipeline Overview` → `[[WireCellSigProc Pipeline Overview]]`, `[[WireCell Clus Pipeline Overview]]`
- `Slicing and Tiling` → `[[OmnibusSigProc]]`, `[[ROI Formation]]`
- `WireCell Clus Pipeline Overview` → `[[WireCell Imaging Pipeline Overview]]`
- `PDVD Imaging Configuration` → `[[WireCell Imaging Pipeline Overview]]`, `[[Slicing and Tiling]]`, `[[Charge Solving]]`

**Wrong See Also link fixed (1 file):**
- `Track Fitting and Calorimetry`: `[[PDVD Signal Processing Configuration]]` → `[[PDVD Detector Parameters]]` (drift/diffusion parameters)

**Symmetric detector↔config links added (3 files):**
- `pdhd Detector Parameters` → `[[PDHD Signal Processing Configuration]]`
- `PDHD Signal Processing Configuration` → `[[pdhd Detector Parameters]]`
- `ADC to Electrons Signal Chain` → `[[pdhd Detector Parameters]]`, `[[PDVD vs PDHD Signal Chain Comparison]]`

## [2026-05-07] ingest | Deghosting algorithms — img/ and clus/ deep dive
Read all deghosting source files: `img/src/InSliceDeghosting.cxx` (897 lines), `img/src/ProjectionDeghosting.cxx` (473 lines), `img/src/ShadowGhosting.cxx` (65 lines), `clus/src/clustering_deghost.cxx` (783 lines), `clus/src/NeutrinoDeghoster.cxx` (621 lines), plus review docs.

Pages created:
- `wiki/Clus Deghosting.md` — ClusteringDeghost + deghost_clusters/segments
- `wiki/Deghosting Comparison.md` — synthesis page comparing all 5 components
- `wiki/source-session-2026-05-07-deghosting.md` — session record

Pages updated:
- `wiki/Imaging Deghosting.md` — rewritten with accurate 3-round InSliceDeghosting structure, source-level detail on ProjectionDeghosting 2-pass algorithm
