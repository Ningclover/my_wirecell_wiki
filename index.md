---
tags: [meta]
updated: 2026-05-07 (deghosting analysis)
---

# WireCell Wiki Index

Content catalog for the WireCell knowledge base. Updated by the LLM on every ingest, filed query, or lint pass.

---

## Sources

Raw source ingest records (`type: file`) and session records (`type: conversation`):

- [[source-sigproc-examination]] — Systematic code examination of `sigproc` (~16,600 lines): bugs, efficiency, algorithm docs. 17/40 bugs fixed, 11/25 efficiency issues fixed.
- [[source-clus-examination]] — All 9 documentation files from `clus/docs/`: full clus pipeline from clustering to neutrino tagging.
- [[source-pdvd-wct-config]] — ProtoDUNE-VD Jsonnet configuration files controlling the actual WCT processing pipeline.
- [[source-session-2026-04-14-clustering-internals]] — Code investigation session: heap corruption debugging in `clustering_extend.cxx`, `Find_Closest_Points` and `get_hull` internals.
- [[source-session-2026-04-25-pdhd-sp-deconvolution]] — Full OmnibusSigProc deconvolution + ROI pipeline walkthrough; PDHD APA0 anomalous Wiener threshold root-cause; three-way config comparison.
- [[source-session-2026-04-26-adc-to-electrons]] — Signal amplitude chain: raw ADC → NF → 2D deconvolution → e⁻/tick; ADC_mV conversion, electronics response normalization, sign convention.
- [[source-session-2026-04-26-pdvd-signal-chain]] — PDVD signal chain analysis: bottom vs top APA electronics, JsonElecResponse units, postgain as effective gain rescaling, comparison to PDHD.
- [[source-session-2026-04-29-track-sim-geometry]] — Sim-track placement for ProtoDUNE-VD/HD per-anode validation: wires-JSON schema, active drift bounds, reframer tick origin, pdhd `elecGain` extVar.
- [[source-session-2026-05-03-pdvd-field-response]] — Full PDVD field response computation pipeline using `pochoir`: 10-stage 3D electrostatics + drift simulation producing `FR_xn_new.json.bz2`.
- [[source-img-examination]] — Systematic code examination of `img/` (~38 source files): pipeline overview, slicing/tiling, charge solving, deghosting, 18 bugs (13 fixed), 15 efficiency issues (7 fixed).
- [[source-session-2026-05-07-deghosting]] — Source-level investigation of all 5 deghosting components across img/ and clus/: multi-round InSliceDeghosting, ProjectionDeghosting, ClusteringDeghost, deghost_clusters, deghost_segments.

---

## Algorithms

Algorithms are organized by toolkit module. Each module has sub-sections matching its internal structure.

### General / Cross-cutting

- [[WireCell Wires Schema]] — JSON wire-geometry format (`Store.{anodes,faces,planes,wires,points}`), list-position vs ident references, per-detector counts.
- [[WireCell Sim Track Conventions]] — Track depo construction (`sim.tracks`, charge=-500 MIP), active-volume placement constraint, reframer tick origin, expected hot-channel counts.

### sigproc

Signal processing module: ADC waveform → deconvolved charge with ROI.

#### Noise Filtering

- [[Omnibus Noise Filter]] — Three-pass noise filtering orchestrator (per-channel, grouped coherent, status)
- [[OmniChannelNoiseDB]] — JSON-configurable per-channel noise parameter and filter spectrum database

#### Signal Processing

- [[WireCellSigProc Pipeline Overview]] — Full ADC→charge pipeline: noise filtering + signal processing architecture
- [[OmnibusSigProc]] — Master signal processing orchestrator (2D deconvolution + ROI pipeline)
- [[ROI Formation]] — Two-tier (tight + loose) region-of-interest identification
- [[ROI Refinement]] — Multi-stage iterative ROI cleanup (BFS, multi-plane, BreakROI, ShrinkROI, ExtendROI)
- [[L1SP Filter]] — LASSO sparse deconvolution for shorted wire regions
- [[Detector-Specific Signal Processing]] — Per-detector noise filtering comparison (MicroBooNE, ProtoDUNE-SP/HD/VD, DuneCrp, ICARUS)

---

### clus

Pattern recognition module: 3D blob cloud → particle trajectories + neutrino vertex.

#### Clustering

- [[WireCell Clus Pipeline Overview]] — IEnsembleVisitor pattern, 6-stage pipeline (clustering stages 1–3, PR stages 4–6), data hierarchy
- [[Clus Data Structures]] — Facade layer, PC tree, PR::Graph/Vertex/Segment/Shower/Fit types
- [[Clustering Algorithms Internals]] — `clustering_regular`, `Find_Closest_Points`, `get_strategic_points`, `get_hull`, `ClusterCache`; crash investigation log

#### Pattern Recognition

- [[Steiner Graph]] — Retiling + k-d neighbor search + Kruskal MST skeleton; degree semantics
- [[Pattern Recognition PR Loop]] — find_proto_vertex() 9-step loop, vertex scoring, SCN DL variant
- [[Track Shower Separation]] — Topology (RMS spread) + trajectory (KS-test) classification, shower grouping, kinematics
- [[Neutrino Vertex Determination]] — Per-cluster scoring, global multi-cluster selection, SCN DL override (2 cm cut)
- [[Track Fitting and Calorimetry]] — Wire→space projection, dQ/dx measurement, Box recombination model, ParticleDataSet
- [[Particle Identification]] — is_shower predicate, determine_direction dispatch, BFS PDG propagation
- [[Neutrino Taggers]] — NuMu, NuE, Pi0, Cosmic, SSM, SinglePhoton criteria and outputs
- [[Clus Deghosting]] — ClusteringDeghost (longest-first 2D kd-tree, single-APA); deghost_clusters (3-cloud segment-level, 4 criteria); deghost_segments (terminal arm pruning)

---

### imaging

3D imaging module: frame → slices → blobs → clusters → charge solving → deghosting.

- [[WireCell Imaging Pipeline Overview]] — component zoo, cluster graph structure, data interfaces, RayGrid, live/dead categories
- [[Slicing and Tiling]] — MaskSlice, SumSlice, GridTiling, BlobGrouping, BlobClustering, GeomClusteringUtil
- [[Charge Solving]] — ChargeSolving, CSGraph LASSO with Cholesky whitening, BlobSolving, ChargeErrorFrameEstimator
- [[Imaging Deghosting]] — InSliceDeghosting 3 rounds (wire-score, charge-ratio, POTENTIAL_GOOD keep), Projection2D, ProjectionDeghosting (2-pass projection coverage), ShadowGhosting (stub)

### charge-light matching *(pending)*

Flash-matching module: optical detector constraints on drift coordinate. Expected after next ingest.

---

## Experiments & Configuration

Configuration files define how the algorithms above are wired and tuned for a specific detector. Detector-specific geometry and electronics parameters live here alongside the run-chain configs.

### ProtoDUNE-VD (PDVD)

- [[PDVD Detector Parameters]] — Geometry, ADC (14-bit), bottom/top electronics, field response, noise spectra, channel layout
- [[ProtoDUNE-VD WireCell Configuration Overview]] — Full pipeline entry points (nf, nf-sp, nf-sp-img), frame tag flow
- [[PDVD Noise Filtering Configuration]] — NF filter assembly: PDVDOneChannelNoise, CoherentNoiseSub, ShieldCouplingSub (top anodes only)
- [[PDVD Signal Processing Configuration]] — OmnibusSigProc per-anode config: thresholds, ctoffset=4µs tied to FR file, ADC conversion
- [[PDVD SP Filters]] — SP filter kernel parameters (ROI LF filters, Gaussian, Wiener, wire filters)
- [[PDVD DNN ROI]] — DNN-based ROI finding: DNNROIFinding for U+V, W shunt, TorchService/TritonService
- [[PDVD Imaging Configuration]] — 3D imaging: pre-proc, slicing (tick_span=4), tiling (GridTiling), charge solving
- [[PDVD Clustering Configuration]] — Per-APA `MultiAlgBlobClustering` pipeline order (Pointed → LiveDead → Extend → Regular×2)
- [[PDVD Field Response Computation]] — 10-stage `pochoir` pipeline: domain → gen → FDM → bc-interp → extendwf → drift → induce → convertfr; produces `FR_xn_new.json.bz2`

### ProtoDUNE-HD (pdhd)

- [[pdhd Detector Parameters]] — Geometry, ADC/electronics, `elecGain` extVar, sim overrides (DL/DT/lifetime/drift_speed), NF intraces wildcard, channel layout
- [[PDHD Signal Processing Configuration]] — SP filter assignments, per-APA parameter tuning, output archive contents, summary_wiener→imaging threshold chain
- [[ADC to Electrons Signal Chain]] — Full amplitude chain: raw ADC → 2D deconvolution → e⁻/tick; ADC_mV conversion, electronics response units, sign convention
- [[PDVD vs PDHD Signal Chain Comparison]] — Side-by-side: electronics type/gain, ADC fullscale, postgain, Resampler, ShieldCouplingSub, ADC/e⁻ sensitivity for bottom/top/PDHD

---

## Synthesis / Meta

- [[SigProc Bug List]] — Consolidated 40-bug list with fix status
- [[SigProc Efficiency Issues]] — Consolidated 25-issue efficiency list with fix status
- [[Img Bug List]] — 18-bug list from `img/` examination (13 fixed, 5 not fixed)
- [[Img Efficiency Issues]] — 15-issue efficiency list from `img/` examination (7 fixed, 8 not fixed); hardcoded constants table for PDHD porting
- [[Deghosting Comparison]] — All 5 deghosting components: pipeline position, granularity, distance thresholds, philosophy, multi-APA handling
