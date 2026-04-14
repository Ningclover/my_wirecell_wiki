---
tags: [meta]
updated: 2026-04-14
---

# WireCell Wiki Index

Content catalog for the WireCell knowledge base. Updated by the LLM on every ingest, filed query, or lint pass.

## Sources

- [[source-sigproc-examination]] — Systematic code examination of `sigproc` (~16,600 lines): bugs, efficiency, algorithm docs. 17/40 bugs fixed, 11/25 efficiency issues fixed.
- [[source-pdvd-wct-config]] — ProtoDUNE-VD Jsonnet configuration files controlling the actual WCT processing pipeline.

## Algorithms

- [[WireCellSigProc Pipeline Overview]] — Full ADC→charge pipeline: noise filtering + signal processing architecture
- [[OmnibusSigProc]] — Master signal processing orchestrator (2D deconvolution + ROI pipeline)
- [[ROI Formation]] — Two-tier (tight + loose) region-of-interest identification
- [[ROI Refinement]] — Multi-stage iterative ROI cleanup (BFS, multi-plane, BreakROI, ShrinkROI, ExtendROI)
- [[L1SP Filter]] — LASSO sparse deconvolution for shorted wire regions
- [[Detector-Specific Signal Processing]] — Per-detector noise filtering (MicroBooNE, ProtoDUNE-SP/HD/VD, DuneCrp, ICARUS)

## Components

- [[Omnibus Noise Filter]] — Three-pass noise filtering orchestrator (per-channel, grouped coherent, status)
- [[OmniChannelNoiseDB]] — JSON-configurable per-channel noise parameter and filter spectrum database

## ProtoDUNE-VD Configuration

- [[ProtoDUNE-VD WireCell Configuration Overview]] — Full pipeline entry points, frame tag flow, design notes
- [[PDVD Detector Parameters]] — Geometry, ADC, electronics (bottom vs top), field response, noise spectra, channel layout
- [[PDVD Noise Filtering Configuration]] — NF filter assembly: PDVDOneChannelNoise, CoherentNoiseSub, ShieldCouplingSub
- [[PDVD Signal Processing Configuration]] — OmnibusSigProc per-anode config: thresholds, timing, ADC conversion, frame tags
- [[PDVD SP Filters]] — SP filter kernel parameters (ROI LF filters, Gaussian, Wiener, wire filters)
- [[PDVD DNN ROI]] — DNN-based ROI finding (DNNROIFinding, TorchService/TritonService, W shunt)
- [[PDVD Imaging Configuration]] — 3D imaging: pre-proc, slicing, tiling, charge solving

## Experiments

*(MicroBooNE, ProtoDUNE-SP, ProtoDUNE-HD standalone pages pending)*

## Synthesis / Meta

- [[SigProc Bug List]] — Consolidated 40-bug list with fix status
- [[SigProc Efficiency Issues]] — Consolidated 25-issue efficiency list with fix status
