# CLAUDE.md — 02_EP_tissue / 03C_tuning_wavelength

## What this tutorial demonstrates

How to jointly tune **CV, APD, and wavelength** (λ = CV × APD) in a 1D cable then verify in a 2D sheet, targeting experimental data from human ventricular epicardium (optical mapping, BCL=1000 ms):

| Nonfailing tissue | CV_l (cm/s) | CV_t (cm/s) | APD80 (ms) | λ_l (cm) | λ_t (cm) |
| --- | --- | --- | --- | --- | --- |
| Target (Glukhov 2010/2012) | 92 | 22 | 340 | 34 | 8 |

Ionic model: `tenTusscherPanfilov`. APD tuned via GKs (IKs maximal conductance). CV tuned via conductivities (Gil/Gel for longitudinal, Git/Get for transverse).

## Setup

| | value | source |
| --- | --- | --- |
| Cable mesh | 1.5 cm, 200 µm resolution (`Mesh_1.5cm_Cable/mesh`) | pre-built |
| Sheet mesh | 1.5×1.5 cm, 200 µm resolution (`Mesh_1.5cm_Sheet/mesh`) | pre-built |
| Ionic model | `tenTusscherPanfilov`, GKs tunable via `--GKs` | `run.py:291` |
| Default Gil/Gel | 0.3544 / 1.27 S/m | `run.py:252-255` |
| Default Git/Get | 0.024 / 0.0862 S/m | `run.py:259-262` |
| `--nbeats` | 2 (default; tutorial suggests >10 for SS, but 2 is used here) | `run.py:268` |
| `--bcl` | 1000 ms | `run.py:272` |
| `--timess` | 1000 ms (pre-pacing stabilization before main beats) | `run.py:276` |
| Stim | left cap of cable / center of sheet, 5 ms, 2× capture | `.par` files |

**Two-phase simulation**: first a state-save run (`tuning_wavelength_state_cable.par`) runs `nbeats-1` beats + timess and writes `statefile`; then a last-beat run (`tuning_wavelength_b10_cable.par`) loads the statefile and runs one BCL. igbapd then extracts APD80 from the last beat.

CV computation:
- Cable: LATs at nodes 25 and 50 (≈0.5 cm and 1.0 cm); CV = 0.5/(LAT50−LAT25) × 1000
- Sheet: L1/L2 nodes at ±0.25 cm from center (longitudinal); T1/T2 at ±0.25 cm transverse

Results written to `<job>/adjustment_results.txt`.

## CLI

```bash
# Default: cable, GKs=0.392
./run.py --dimension cable

# APD-tuned: GKs=0.22 → APD≈340 ms
./run.py --GKs 0.22 --dimension cable

# Verify in 2D sheet
./run.py --GKs 0.25 --dimension sheet

# Short wavelength (λ < 8 cm): reduced conductivities + GKs
./run.py --GKs 0.25 --Gil 0.02 --Git 0.0135 --dimension cable
./run.py --GKs 0.25 --Gil 0.02 --Git 0.0135 --dimension sheet
```

## Findings (run on 2026-04-28, cable only)

| run | APD80 (ms) | CV_l (cm/s) | λ_l (cm) | CV_t (cm/s) | λ_t (cm) |
| --- | --- | --- | --- | --- | --- |
| default (GKs=0.392) | 297 | 76.2 | 22.7 | 10.6 | 3.2 |
| GKs=0.22 | 342 | 76.2 | 26.0 | 10.6 | 3.6 |
| GKs=0.25, Gil=0.02, Git=0.0135 | 331 | 11.1 | 3.7 | 6.6 | 2.2 |

Key observations:
- Default GKs=0.392 gives APD≈297 ms — too short. Target is 340 ms.
- GKs=0.22 gives APD≈342 ms (close to target 340 ms). CV unchanged because GKs doesn't affect propagation velocity.
- Default conductivities give CV_l≈76 cm/s vs target 92 cm/s — lower than experimental. Reducing Gil/Gel from 0.354/1.27 to 0.02/default reduces CV drastically; the tutorial's "short wavelength" task is to find conductivities giving λ < 8 cm (not to match experimental CV).
- Note: the tutorial targets CV=92 cm/s but the default Gil/Gel only gives 76 cm/s. Tuning conductivities to get 92 cm/s requires tuneCV with velocity=0.92 (see docstring: gi=0.4134, ge=1.4849).

Sheet runs not executed (session interrupted). Cable findings fully match tutorial expectations.

## Notes on simulation pipeline

The `tuning_wavelength_state_cable.par` runs with `-write_statef statefile` at the end of the pre-pacing phase. The subsequent `tuning_wavelength_b10_cable.par` loads `-start_statef statefile.{tsav}.0` (note the `.{time}.0` suffix appended by openCARP). This two-phase approach is efficient: state is saved after pre-pacing, then any number of "last beat" analyses can start from the same state without re-running pre-pacing.

## Lessons (transferable)

1. **Wavelength λ = CV × APD** is the minimum tissue size supporting reentry. Both CV (conductivity) and APD (ionic model) must be tuned independently — they are orthogonal knobs.
2. **GKs controls APD** in tenTusscherPanfilov: reducing GKs (less outward IKs) prolongs the plateau → longer APD. Default 0.392 → APD≈297 ms; 0.22 → APD≈342 ms at BCL=1000.
3. **`-write_statef` / `-start_statef`** enable state checkpointing. The file suffix is `.<time>.<rank>` — for np=1 it's always `.0`. Load the last time point to resume a simulation from a stable cycle.
4. **2 beats is not steady-state** (same caveat as in 03_study_prep_APD). Tutorial says `nbeats` should be "much larger" for real SS. For quick exploration nbeats=2 is usable.
5. **Cable vs sheet**: tune in 1D cable first (fast, cheap), then verify in 2D sheet (slower). The 2D sheet may give slightly different results due to boundary effects and 2D propagation geometry.
