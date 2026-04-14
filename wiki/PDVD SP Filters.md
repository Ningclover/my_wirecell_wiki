---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD SP Filters

Signal processing filter parameters for ProtoDUNE-VD, defined in `sp-filters.jsonnet`. These are the "optimized May 2019" parameters. The file warns that filter names are hard-coded in the C++ SP code — do not change instance names.

## ROI low-frequency filters (LfFilter: high-pass)

Shape: `1 - exp(-(freq/tau)²)`, suppresses below tau.

| Name | tau | Note |
|------|-----|------|
| `ROI_tight_lf` | 0.014 MHz | Tightest (was 0.02) |
| `ROI_tighter_lf` | 0.06 MHz | Medium (was 0.1) |
| `ROI_loose_lf` | 0.002 MHz | Loosest (was 0.0025) |

## Gaussian filters (HfFilter: low-pass)

Shape: `exp(-0.5 * (freq/sigma)^power)`.

| Name | sigma | power | Note |
|------|-------|-------|------|
| `Gaus_tight` | 0 | 2 | No smearing (flat response) |
| `Gaus_wide` | 0.12 MHz | 2 | Wide Gaussian smearing |

## Wiener filters per plane (HfFilter: low-pass)

Applied within ROIs for optimal S/N. Two widths per plane (tight/wide).

| Name | sigma (MHz) | power |
|------|-------------|-------|
| `Wiener_tight_U` | 0.148788 | 3.76194 |
| `Wiener_tight_V` | 0.1596568 | 4.36125 |
| `Wiener_tight_W` | 0.13623 | 3.35324 |
| `Wiener_wide_U` | 0.186765 | 5.05429 |
| `Wiener_wide_V` | 0.1936 | 5.77422 |
| `Wiener_wide_W` | 0.175722 | 4.37928 |

## Wire-direction filters (HfFilter in wire space)

Shape: `exp(-0.5 * (wire/sigma)^2)`, where sigma is in wire-number space.

| Name | sigma | Note |
|------|-------|------|
| `Wire_ind` | 1/√π × 5.0 ≈ 2.82 | Induction (was ×1.4, then ×0.75) |
| `Wire_col` | 1/√π × 10.0 ≈ 5.64 | Collection (wider, accommodates broader collection response) |

## Notes

- All filters have `max_freq = 1 MHz` (Nyquist for 500 ns tick)
- Wiener filters use `flag: true` (with DC suppression); wire filters use `flag: false`
- Commented-out block shows the pre-May-2019 defaults (e.g., Wire_ind was 1/√π×1.4, Wire_col was ×3.0)

## See also

[[PDVD Signal Processing Configuration]], [[OmnibusSigProc]]

## Sources

- [[source-pdvd-wct-config]]
