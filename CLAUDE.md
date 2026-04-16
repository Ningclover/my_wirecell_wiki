# WireCell Wiki Schema

This is a personal knowledge base about **WireCell Toolkit** — the LArTPC signal processing and reconstruction framework used by MicroBooNE, ICARUS, SBND, DUNE, and related experiments. Maintained by an LLM agent.

You (Claude) own the `wiki/` directory entirely. You read from `raw/` but never modify it. The human curates sources and asks questions; you do the writing, cross-referencing, and maintenance.

## Directory layout

```
raw/          — immutable source documents (papers, code notes, talks, meeting notes)
raw/assets/   — locally downloaded images referenced by source files
wiki/         — all LLM-generated markdown pages
wiki/index.md — content catalog (update on every ingest)
wiki/log.md   — append-only chronological log (update on every operation)
CLAUDE.md     — this file (schema and operating instructions)
```

## Wiki page conventions

- Every page lives in `wiki/` as a plain markdown file.
- **Filename must exactly match the wikilink text** so Obsidian can resolve links. Use Title Case with spaces. E.g. `ROI Formation.md`, `OmnibusSigProc.md`, `PDVD SP Filters.md`.
- Exception: source/meta pages use lowercase-hyphen: `source-*.md`, `index.md`, `log.md`.
- Every page starts with a level-1 heading matching the page title.
- Use `[[Page Title]]` Obsidian-style wikilinks for cross-references. The link text must match the filename (without `.md`).
- YAML frontmatter for Dataview queries:
  ```yaml
  ---
  tags: [algorithm]
  sources: 2
  updated: 2026-04-14
  ---
  ```

### Source page types

Two kinds of source pages exist, both using lowercase-hyphen filenames:

**File-based source** (`source-<slug>.md`) — created when ingesting a raw file or directory. Standard format: frontmatter with `type: file`, summary of what was read, pages created.

**Session record** (`source-session-YYYY-MM-DD-<slug>.md`) — created when knowledge comes from a debugging or investigation conversation rather than a raw file. This is the citable anchor for conversation-derived knowledge. Format:

```markdown
---
tags: [source]
type: conversation
date: YYYY-MM-DD
context: <what was being investigated>
files_touched: [src/foo/Bar.cxx, src/foo/Baz.cxx]
updated: YYYY-MM-DD
---

# source-session-YYYY-MM-DD-<slug>

Brief description of the session.

## Confirmed findings
- <finding> — see [[Page Title]]

## Context
<What problem was being debugged, what experiment/module.>
```

Session records sync via GitHub alongside wiki pages and serve as `Sources:` citations exactly like file-based source pages. Add `type: file` to existing file-based source pages for consistency.

### Wiki top-level structure

`index.md` is organized into four sections. Place new pages in the correct section:

1. **Sources** — `source-*.md` ingest records (one per raw source directory) and `source-session-*.md` session records (one per debugging/investigation conversation).
2. **Algorithms** — algorithm and component pages, grouped by toolkit module:
   - `sigproc`: sub-sections "Noise Filtering" and "Signal Processing"
   - `clus`: sub-sections "Clustering" and "Pattern Recognition"
   - `imaging`: (pending) sub-sections TBD
   - `charge-light matching`: (pending) sub-sections TBD
   - Add a new module section when a new raw module is ingested.
3. **Experiments & Configuration** — per-experiment detector parameters + pipeline wiring configs:
   - `PDVD`: ProtoDUNE-VD pages
   - `pdhd`: ProtoDUNE-HD pages (pending)
   - Add a new sub-section when a new experiment config is ingested.
4. **Synthesis / Meta** — cross-source analysis, bug lists, efficiency summaries.

### Categories and tags

- **algorithm** — a specific reconstruction or signal processing algorithm (e.g. Wire-Cell Imaging, ROI finding, noise filtering, tomographic reconstruction)
- **component** — a WireCell Toolkit software component, package, or plugin (e.g. `WireCellSigProc`, `WireCellImg`, `WireCellGen`)
- **concept** — a physics or detector concept underlying WireCell (e.g. LArTPC, induction signal, bipolar pulse, wire plane, drift field)
- **experiment** — a detector or experiment that uses WireCell (e.g. MicroBooNE, ICARUS, SBND, ProtoDUNE, DUNE)
- **person** — a key contributor or collaborator
- **source** — summary of a single raw source document (paper, talk, note)
- **synthesis** — cross-source analysis, comparison table, or discovered connection
- **meta** — index, log, and other housekeeping pages

### Suggested page structure for algorithms

```
# Algorithm Name
Brief one-line description.

## Purpose
What problem it solves.

## Method
How it works (key steps, math if relevant).

## Inputs / Outputs
What it takes and produces.

## Configuration
Key parameters and their effect.

## Known issues / limitations

## See also
[[Related Page]], [[Other Page]]

## Sources
- [[source-foo]]
```

## Operations

### Ingest
When the user asks you to ingest a source (`raw/<file>`):
1. Read the source file (and any referenced images if relevant).
2. Discuss key takeaways with the user if they want.
3. Create a `wiki/source-<slug>.md` summary page.
4. Update or create algorithm, component, concept, and experiment pages touched by the source.
5. Update `wiki/index.md` — add the new source page and any new pages.
6. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <Title>`.

### Query
When the user asks a question:
1. Read `wiki/index.md` to find relevant pages.
2. Read those pages and synthesize an answer with citations (`[[Page Title]]`).
3. If the answer is valuable, offer to file it as `wiki/synthesis-<slug>.md`.
4. If filed, update `wiki/index.md` and append to `wiki/log.md`.

### Re-link
When the user asks for a re-link pass (e.g., after a `git pull` that brought in pages from another server):
1. Read `wiki/index.md` to collect all known page titles.
2. For each wiki page, scan its body for unlinked mentions of known page titles and add `[[wikilinks]]`.
3. Scan `See also` sections for missing obvious cross-references and add them.
4. Fix any broken wikilinks pointing to non-existent pages (de-link or redirect to the correct page).
5. Ensure every algorithm/component page has a `## Sources` section.
6. Append `## [YYYY-MM-DD] relink` to `wiki/log.md`.

### Lint
When the user asks for a wiki health check:
- Look for: contradictions, stale claims, orphan pages, concepts mentioned but lacking their own page, missing cross-references, algorithms with no known-issues section.
- Suggest papers or docs to look for that could fill gaps.
- Append `## [YYYY-MM-DD] lint` to `wiki/log.md`.

## Style guidelines

- Be concise. Prefer bullet points over prose.
- Every algorithm/component page should have a **Sources** section.
- Flag contradictions: `> **Contradiction:** ...`
- Flag uncertainty: `(unverified)` inline.
- Do not invent facts — only write what is supported by raw sources or the user's statements.
- For code references, use backtick paths: `` `src/sigproc/Microboone.cxx` ``.
