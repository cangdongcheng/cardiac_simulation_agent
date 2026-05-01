# CLAUDE.md — 02_EP_tissue / 03E_study_resolution

## What this tutorial demonstrates

The **Niederer N-version benchmark** (Niederer et al. 2011): a standard EP test problem for verifying cardiac tissue simulators and assessing spatial convergence. A 20×7×3 mm cuboid is stimulated at the (0,0,0) corner and activation times along the body diagonal are compared across mesh resolutions (500, 200, 100 µm).

## Setup

| | value | source |
| --- | --- | --- |
| Geometry | 20×7×3 mm cuboid | `run.py:238` |
| Ionic model | `tenTusscherPanfilov` (EPI variant) | `nversion.par` |
| Initial state | `singlecell.sv` (pre-paced limit cycle) | `nversion.par` |
| Stimulus | 50,000 µA/cm³, 2ms, box [0,0,0]→[1500,1500,1500] µm | `nversion.par` |
| Conductivities | g_il=0.17, g_it=0.019, g_el=0.62, g_et=0.24 S/m | `nversion.par` |
| Fiber direction | Longitudinal along X axis | `run.py` |
| Threshold | Vm ≥ 0 mV for activation | `init_acts_vm_act-thresh.dat` |
| dt | 20 µs | `--dt 20` |
| Mass lumping | Off (default) | `--massLumping 0` |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/03E_study_resolution

# Run the full benchmark (all 3 resolutions)
PATH=/home/cdc/miniconda3/envs/opencarp/bin:/usr/local/lib/opencarp/lib/petsc/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py --compare --np 4

# Single resolution
PATH=... python run.py --dx 100 --tend 50 --np 4
PATH=... python run.py --dx 200 --tend 70 --np 4
PATH=... python run.py --dx 500 --tend 150 --np 4

# With mass lumping (shows larger error)
PATH=... python run.py --compare --massLumping 1 --np 4
```

## Diagonal node selection

The benchmark compares activation times at nodes along the body diagonal from (0,0,0) to (20,7,3) mm. The code finds the maximum `diag_divs` such that all three dimension division counts (`divs = point_dims - 1`) are divisible by `diag_divs`, giving a common grid of equally-spaced points in all three axes simultaneously.

| dx | point_dims | divs | diag_divs | diagonal nodes |
| --- | --- | --- | --- | --- |
| 100 µm | [201, 71, 31] | [200, 70, 30] | 10 | 11 points |
| 200 µm | [101, 36, 16] | [100, 35, 15] | 5 | 6 points |
| 500 µm | [41, 15, 7] | [40, 14, 6] | 2 | 3 points |

## Findings (run on 2026-04-28)

All 3 resolutions ran successfully with `--compare --np 4`.

### Activation times along body diagonal (full mass matrix, no lumping)

Diagonal length = √(20² + 7² + 3²) = 21.40 mm

| dist (mm) | dx=100 µm | dx=200 µm | dx=500 µm |
| --- | --- | --- | --- |
| 0.00 | 1.278 ms | 1.278 ms | 1.276 ms |
| 2.14 | 2.525 ms | — | — |
| 4.28 | 6.020 ms | 6.119 ms | — |
| 8.56 | 14.529 ms | 14.626 ms | — |
| 10.70 | 18.983 ms | — | 25.009 ms |
| 12.84 | 23.463 ms | 23.574 ms | — |
| 17.12 | 32.432 ms | 32.606 ms | — |
| 21.40 | **40.654 ms** | **40.865 ms** | **58.329 ms** |

### Key convergence result

- **100 µm vs 200 µm**: far-corner activation time differs by 0.21 ms (0.5%) — essentially converged
- **100 µm vs 500 µm**: far-corner activation time differs by 17.7 ms (43.5%) — gross error at coarse resolution
- The 500 µm mesh significantly overestimates the activation time (wave travels slower on a coarse grid)

### Why coarse meshes are too slow

On a coarse mesh, the spatial gradient of Vm is poorly resolved near the wavefront. The limited coupling between neighboring nodes means less depolarizing current diffuses ahead — the wavefront advances slower in the simulation even though the "true" CV should be higher. This is a finite-element artifact, not a physiological effect.

### Simulation run times (np=4)

| dx | nodes | tend | wall time (approx) |
| --- | --- | --- | --- |
| 100 µm | 442,401 | 50 ms | ~40 min |
| 200 µm | 58,176 | 70 ms | ~5 min |
| 500 µm | 4,305 | 150 ms | ~1 min |

## Key rules to remember

1. **Resolution requirement**: For accurate CV and APD, use ≤200 µm; 100 µm gives essentially grid-converged results for the tenTusscherPanfilov model.
2. **Mass lumping**: The tutorial (and original benchmark) shows that mass lumping greatly worsens spatial-resolution-dependence. Avoid unless explicitly needed for stability.
3. **tend must exceed the activation time at the far corner**: 50ms is sufficient for 100µm (far corner activates ~40ms); 70ms for 200µm; 150ms for 500µm (far corner ~58ms).
4. **The `--compare` flag runs all 3 resolutions sequentially**, sharing all other args except `--dx` and `--tend`.

## Lessons (transferable)

1. **Always convergence-test EP models** at a new spatial resolution before trusting quantitative results.
2. **Activation time is the primary observable** for spatial convergence checks; voltage traces at fixed points are also useful.
3. **The Niederer benchmark is the community standard** for verifying new EP solver implementations. OpenCARP results match expectations from Niederer 2011 (< 1% difference between 100 and 200 µm).
4. `init_acts_vm_act-thresh.dat` stores one activation time per mesh node; `retagged.dat` is different (per-element tags).
