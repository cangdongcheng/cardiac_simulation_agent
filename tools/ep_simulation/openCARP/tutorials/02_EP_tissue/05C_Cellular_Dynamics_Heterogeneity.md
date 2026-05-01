# CLAUDE.md — 02_EP_tissue / 05C_Cellular_Dynamics_Heterogeneity

## What this tutorial demonstrates

Region-wise assignment of **ionic model parameters** (channel conductances) to four horizontally-striped regions of a 1 cm × 1 cm 2D sheet, showing how changes in sodium or potassium channel conductances affect conduction velocity and repolarization. Uses the same pre-built mesh as 05B but modulates the ionic model instead of conductivity.

Ionic model: **UCLA_RAB** (MahajanShiferaw). Four stripes from bottom to top get a selected channel conductance scaled by ×1/×2/×4/×8 (up) or ×1/÷2/÷4/÷8 (down).

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | `TestSlabMesher2D_i.*` (coupled) or `TestSlabMesher2D_i_split.*` | shipped |
| Ionic model | MahajanShiferaw | `Ionic-Variability.par` |
| Variable choices | GNa, GKs, GKr, GK1, Gss, Gtof, Gtos | `--variable` |
| Scaling | ×1/2/4/8 (up) or ×1/÷2/÷4/÷8 (down) bottom→top | `run.py:341-344` |
| tend | 80 ms (or 300 ms with `--show_vm`) | `run.py:353` |
| Stim | `LeftSideStim.vtx` left-edge planar | par file |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/05C_Cellular_Dynamics_Heterogeneity
# Default: GNa upscaled
python run.py

# GNa downscaled
python run.py --tune_down

# Different variable
python run.py --variable GKr
python run.py --variable GKs --tune_down
```

## Findings (run on 2026-04-28)

Both runs complete in ~40 s. CV measured from activation time at the right edge (10 mm from stim site).

### GNa upscaled (×1/2/4/8, bottom→top) — default

| Region | GNa scale | CV (cm/s) |
| --- | --- | --- |
| reg0 (bottom) | ×1 (control) | 98 |
| reg1 | ×2 | 116 |
| reg2 | ×4 | 135 |
| reg3 (top) | ×8 | 148 |

GNa↑ accelerates the upstroke → faster conduction. CV increases monotonically but not linearly (×8 GNa gives only 1.5× CV).

### GNa downscaled (×1/÷2/÷4/÷8, bottom→top) — `--tune_down`

| Region | GNa scale | CV (cm/s) |
| --- | --- | --- |
| reg0 (bottom) | ×1 (control) | 89 |
| reg1 | ÷2 | 77 |
| reg2 | ÷4 | 60 |
| reg3 (top) | ÷8 | 43 |

GNa↓ slows conduction monotonically. No conduction block at ÷8 (GNa channel still present; compare with conductivity reduction where source-sink changes can block). CV at ÷8 is ~43 cm/s — significantly below control but still propagating.

### Contrast with conductivity heterogeneity (05B)

- **Conductivity** (05B): giL×64 range → CV ratio ~3.7× (99 vs 27 cm/s). Highly nonlinear.
- **GNa** (05C): GNa×8 range → CV ratio ~1.5× (148 vs 98 cm/s). More compressed.

This reflects that CV depends on multiple factors (GNa affects upstroke rate, conductivity affects source-sink geometry). GNa modulates the activation rate of the leading-edge wavefront; conductivity modulates how much current diffuses to depolarize downstream cells.

## Lessons (transferable)

1. **`-imp_region[N].im_param`** applies a modifier string for the tagged region. Format: `GNa*2,GKs/0.5` etc. Verified in `parameters.par` under `imp_region[N].im_param`.
2. **GNa effect on CV**: doubling GNa gives ~18% faster CV in this model at these conductivities; quadrupling gives ~38%. Saturation behavior — there's a ceiling because the diffusion-limited part of conduction is unchanged.
3. **Ionic modulation vs. conductivity modulation**: both change CV but via different mechanisms. Ionic modulation (GNa) changes the upstroke rate; conductivity changes the electrotonic coupling length. For arrhythmia studies, knowing which knob is more physiologically relevant matters.
4. **`--show_vm` runs tend=300 ms** to capture full AP cycle including repolarization — allows visualization of APD differences between regions when potassium channels are modulated.
5. **Coupling effect**: in the non-split mesh, the higher-CV upper regions partially "drag" the lower regions. To see each region's intrinsic CV, use `--split` (matched with `TestSlabMesher2D_i_split.*` + `LeftSideStim_split.vtx`).
