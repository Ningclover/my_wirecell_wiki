---
tags: [algorithm, component]
sources: 1
updated: 2026-04-14
---

# L1SP Filter

L1-norm regularized sparse signal deconvolution for **shorted wire regions** where induction and collection plane wires are electrically connected. Standard deconvolution fails in these regions; L1SP jointly decomposes the mixed signal.

**File:** `src/L1SPFilter.cxx` (692 lines)  
**Header:** `inc/WireCellSigProc/L1SPFilter.h`

## Purpose

In shorted wire regions, the measured ADC waveform is a superposition of both collection-plane (unipolar) and induction-plane (bipolar) responses. L1SP uses LASSO (compressed sensing) to separate them.

## Mathematical formulation

```
minimize  ‖G·β − W‖₂²  +  λ·‖β‖₁
```

- **W** (size N): observed ADC waveform in the ROI
- **β** (size 2N): unknown signal split into W-plane (β[0..N-1]) and V-plane (β[N..2N-1]) components
- **G** (N × 2N): forward model encoding both detector response functions
- **λ**: L1 regularization strength (enforces sparsity — physically expected for wire signals)

## Processing pipeline

1. **ROI identification** — from deconvolved signal (positive charge ticks) + raw ADC (`raw_ROI_th_nsigma × σ` threshold), merged with gap tolerance `roi_pad`
2. **First L1 pass** — call `L1_fit` per ROI. Returns: 0 (no action), 1 (shorted, L1 applied), 2 (remove signal)
3. **Forward propagation** — if a flag-1 ROI is found, adjacent flag-2 ROIs within 20 ticks are re-fitted as shorted
4. **Reverse propagation** — same logic scanning in reverse (but see BUG-L1-1 below)
5. **Cross-channel cleaning** — flag-2 ROIs zeroed if neighboring channels (±1) have flag-1 ROI overlapping within 3 ticks
6. **Output** — packaged into output frame with configured tag

## L1_fit classification

- **flag=1** (apply L1): `sum / rescaled_abs_sum > ratio_threshold` AND `abs_sum > sum_threshold` (significant net-positive signal)
- **flag=2** (zero out): rescaled absolute sum below limit, or large decon signal with small raw ADC

## Key configuration parameters

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `l1_seg_length` | 120 | Max ticks per LASSO solve segment |
| `l1_scaling_factor` | 500 | Scales response matrix for numerical conditioning |
| `l1_lambda` | 5 | LASSO regularization strength |
| `l1_epsilon` | 0.05 | LASSO convergence tolerance |
| `l1_niteration` | 100000 | Max LASSO iterations |
| `l1_col_scale` | 1.15 | Weight for collection plane component |
| `l1_ind_scale` | 0.5 | Weight for induction plane component |
| `l1_decon_limit` | 100 | Min signal threshold (electrons) |
| `raw_ROI_th_nsigma` | 4 | N-sigma threshold for raw ROI detection |
| `roi_pad` / `raw_pad` | 3 / 15 | ROI extension padding (ticks) |
| `collect_time_offset` | 3.0 µs | Collection plane time offset relative to induction |

## Known issues / bugs

- **BUG-L1-1** (FIXED): Reverse-scan loop uses `rois_save.size()-1-i1` for distance but forward index `i1` for flag lookup and `L1_fit`. Entire reverse propagation was operating on wrong ROIs. Fixed: introduce `ri = rois_save.size()-1-i1` and use consistently.
- **BUG-L1-2** (FIXED): Division by zero in flag classification when `temp1_sum == 0` (left operand of `&&` evaluated before right). Fixed: swapped operands so the threshold check short-circuits first.
- **BUG-L1-3** (NOT FIXED): `ntot_ticks` used before fully computed — early traces may see incomplete value.
- **BUG-L1-4** (NOT FIXED): Neighbor channel flag map accessed without existence check — `.at(i3)` can throw on empty vector.
- **BUG-L1-5** (NOT FIXED): Raw pointer ownership for `lin_V`/`lin_W` interpolators. Should be `unique_ptr`.
- **BUG-L1-6** (INFO): `m_period` member set but never read — dead state.
- **EFF-L1-1** (FIXED): JSON config re-parsed (~20 `get()` calls) on every `L1_fit` invocation. All values now cached in `configure()`.
- **EFF-L1-2** (NOT FIXED): Dense matrix (up to 120×240) allocated fresh per LASSO segment.
- **EFF-L1-5** (FIXED): Smearing vector parsed from JSON per call. Now cached in `configure()`.
- **EFF-L1-6** (INFO): Default `l1_niteration=100000` — no reporting if solver hits limit without convergence.

## See also

[[WireCellSigProc Pipeline Overview]], [[OmnibusSigProc]]

## Sources

- [[source-sigproc-examination]]
