# CLAUDE.md — 02_EP_tissue / 05B_Conductive_Heterogeneity

## What this tutorial demonstrates

Region-wise assignment of **conductivity** (not ionic parameters) to four horizontally-striped tissue regions on a 1 cm × 1 cm 2D sheet, exploring how different intracellular longitudinal conductivities (gi_L) in each stripe produce different conduction velocities. Three sub-experiments: coupled (default), electrically isolated (`--split`), point stimuli (`--non_planar_stim`).

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | pre-built `TestSlabMesher2D_i.*` (coupled) or `TestSlabMesher2D_i_split.*` (isolated) | shipped files |
| Ionic model | **DrouhardRoberge** (from `CV-Variability.par`) | par file |
| Region conductivities | reg0 (bottom): giL=2.0, reg1: giL=0.5, reg2: giL=0.125, reg3 (top): giL=0.03125 S/m | defaults |
| giT per region | giL / anisotropy_factor (default=4) | `run.py` |
| Stim | vertex file `LeftSideStim.vtx` (planar) or `LeftSideStimWithGaps.vtx` (point) | args |
| tend | 50 ms | `CV-Variability.par` |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/05B_Conductive_Heterogeneity
# Default: coupled, planar stim
python run.py

# Isolated regions (each region paced independently)
python run.py --split

# Point stimuli to see ellipsoidal wavefronts
python run.py --non_planar_stim
```

## Findings (run on 2026-04-28)

Both runs complete in ~25 s.

### CV per region — coupled (`nosplit_planarstim`)

| Region | giL (S/m) | Activation at right edge (ms) | CV (cm/s) |
| --- | --- | --- | --- |
| reg0 (bottom) | 2.000 | 10.2 | **98** |
| reg1 | 0.500 | 12.2 | **82** |
| reg2 | 0.125 | 17.1 | **59** |
| reg3 (top) | 0.03125 | 28.7 | **35** |

In the coupled case, the wavefront from the fast bottom region partially drives the slower upper regions (source-sink coupling), elevating CV in reg1-3 vs. their intrinsic values.

### CV per region — isolated (`split_planarstim`)

| Region | giL (S/m) | Activation at right edge (ms) | CV (cm/s) | Tutorial expected |
| --- | --- | --- | --- | --- |
| reg0 (bottom) | 2.000 | 10.1 | **99** | 102.7 |
| reg1 | 0.500 | 12.8 | **78** | 77.7 |
| reg2 | 0.125 | 20.1 | **50** | 47.5 |
| reg3 (top) | 0.03125 | 36.4 | **27** | 25.6 |

Isolated CVs match tutorial docstring values within ~5%. The tutorial's docstring was likely measured at a slightly different resolution or BCs.

### Key insight: CV ∝ √giL doesn't hold

The ratio of giL between adjacent regions is 4× (2.0→0.5→0.125→0.03125). If CV ∝ √giL:
- Predicted CV ratio: √4 = 2×
- Measured ratio: 99/78 ≈ 1.27, 78/50 ≈ 1.56, 50/27 ≈ 1.85

The √σ approximation from the linear core conductor model is not accurate for finite-element bidomain simulations.

## Lessons (transferable)

1. **`-gregion[N].g_il / g_it`**: the per-region conductivity fields. `g_il` is longitudinal (along fibres), `g_it` transverse. Both are in S/m (= mS/mm). In this tutorial, fibres are uniformly left-to-right.
2. **Split mesh**: `TestSlabMesher2D_i_split.*` has electrical gaps between regions (high-resistance or no connections at region boundaries). Used to measure "native" CV without inter-region coupling effects.
3. **Vertex files**: `LeftSideStim.vtx` = full left-edge planar stimulus. `LeftSideStimWithGaps.vtx` = one stim node per region only (to trigger ellipsoidal wavefronts). Opened with `-stim[0].elec.vtx_file`.
4. **giT = giL/anisotropy_factor**: the default factor of 4 means transverse is 4× slower. This creates ellipsoidal wavefronts visible with point stimuli.
