---
tags: [component, algorithm]
sources: 1
updated: 2026-04-14
---

# Omnibus Noise Filter

Top-level noise filtering orchestrator. Runs three passes over a raw ADC frame to remove per-channel and correlated noise before signal processing.

**File:** `src/OmnibusNoiseFilter.cxx` (294 lines)  
**Header:** `inc/WireCellSigProc/OmnibusNoiseFilter.h`

## Purpose

Convert raw ADC frame → clean ADC frame by applying configurable chains of per-channel and grouped filters. Feeds [[OmnibusSigProc]].

## Three-pass architecture

### Pass 1: Per-channel filters (`m_perchan`)
Applied independently to each channel. Typical operations:
- Stuck-bit / sticky-code mitigation (ProtoDUNE)
- Chirp noise detection (MicroBooNE)
- RC undershoot correction (spectral division by RC response)
- Spectral noise mask (notch filters for known noise frequencies)
- Robust baseline subtraction
- Noisy channel identification (RMS-based)

### Pass 2: Grouped filters (`m_multigroup_chanfilters`)
Applied to groups of channels sharing electronics (same ASIC/motherboard):
- Waveforms copied into `channel_signals_t` map
- Filter computes median waveform across group (with signal protection)
- Scaled subtraction: per-channel correlation coefficient × median
- Results copied back to frame

### Pass 3: Per-channel status (`m_perchan_status`)
Final channel quality determination (e.g., noisy channel tagging).

Bad channels from noise database are zero-filled before processing.

## Related component: PMT Noise Filter

[[PMT Noise Filter]] (`OmnibusPMTNoiseFilter`) removes cross-talk from photomultiplier tubes:
1. Identify PMT ROIs on collection wires (large negative excursions)
2. Identify associated induction wire signals
3. Replace affected regions with linear interpolation

## Known issues / bugs

- **BUG-NF-1** (FIXED): `PMT_ROIs` was a file-scope global vector — not thread-safe, leaked across calls. Moved to member variable `m_pmt_rois`.
- **BUG-NF-2** (FIXED): `min` variable never updated in PMT peak-finding loop — reported peak was last sample below start, not true minimum.
- **BUG-NF-3** (FIXED): Division by zero in `RemovePMTSignal` when `end_bin == start_bin`.
- **BUG-NF-6** (NOT FIXED): If any one channel in a group is missing from the input frame, the **entire group** is skipped for coherent noise subtraction. May be intentional.
- **BUG-NF-7** (FIXED → EFF-NF-7): O(N) linear search for bad channels replaced with `unordered_set`.
- **EFF-NF-1** (FIXED): PMT filter double-copied waveforms. Now stored once, RMS computed from stored copy.
- **EFF-NF-2** (FIXED): Range-for loops copied `shared_ptr`s and vectors. Changed to `const auto&`.
- **EFF-NF-3** (Partial): Grouped filter still copies waveforms in/out of `channel_signals_t` map (required by filter API).

## See also

[[WireCellSigProc Pipeline Overview]], [[OmniChannelNoiseDB]], [[Detector-Specific Signal Processing]]

## Sources

- [[source-sigproc-examination]]
