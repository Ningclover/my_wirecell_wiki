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
