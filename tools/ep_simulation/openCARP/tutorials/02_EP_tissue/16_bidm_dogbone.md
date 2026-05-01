# CLAUDE.md — 02_EP_tissue / 16_bidm_dogbone

## What this tutorial demonstrates

The **dogbone polarization pattern** — a complex virtual-electrode pattern that arises from unequal anisotropy ratios (g_il/g_it ≠ g_el/g_et) under a strong extracellular point stimulus. Instead of the elliptic depolarization/hyperpolarization pattern expected in isotropic tissue, two lobes of opposite polarity appear at 45° to the fiber axis, giving the characteristic "dogbone" shape. This is a purely bidomain phenomenon: monodomain cannot reproduce it.

## Setup

| | value | source |
| --- | --- | --- |
| Geometry | 20×20×10 mm 3D hexahedral mesh + 5 mm bath above/top | auto-generated |
| Ionic model | Plonsey (passive membrane) | par flag |
| Bidomain | yes (`bidomain=1`) | run.py |
| Stimulus | extracellular point source (anode or cathode) at top bath, `crct.type=1` | run.py |
| Ground | flat electrode at bottom of bath, `crct.type=3` | run.py |
| Stim strength | −1×10⁸ µA/cm² (anode default) | parameters.par |
| Duration | 8 ms stim, tend=10 ms total | par |
| Fiber rotation | −60° to +60° transmurally | run.py |
| Resolution | 0.5 mm (default; range 0.1–0.7 mm) | `--res` |

### Conductivity (anisotropy ON — dogbone case)

| Region | g_il | g_it | g_el | g_et |
| --- | --- | --- | --- | --- |
| Myocardium | 0.174 S/m | 0.019 S/m | 0.625 S/m | 0.236 S/m |
| Bath | — | — | 0.625 S/m | 0.625 S/m (isotropic) |

Anisotropy ratio: intracellular g_il/g_it = 9.2, extracellular g_el/g_et = 2.65. **Unequal** (they differ), which is the trigger for dogbone vs. elliptic.

### Conductivity (anisotropy OFF — elliptic case, `--no_anisotropy`)

All conductivities set to 0.625 S/m (isotropic). Result is a symmetric elliptic polarization.

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/16_bidm_dogbone
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py --res 0.5 --np 1

# Cathodal instead of anodal
python run.py --res 0.5 --np 1 --pt_stim cathode

# Isotropic (elliptic pattern)
python run.py --res 0.5 --np 1 --no_anisotropy
```

**Constraints:**
- `--res` must be in [0.1, 0.7] (assertion in run.py). Default 0.5.
- `--np 1` required on this system (no `mpiexec`). The tutorial recommends `--np >2` for speed; at `--np 1` and `--res 0.5` compute time is ~255 s.

## Findings (run on 2026-04-28)

Run: `2026-04-28_dogbone_anode_res0p5_aniso_on`. Anodal stimulus, 0.5 mm resolution, anisotropy ON.

- **Electrics compute**: 255.21 s (init 1.31 s)
- **Ionics compute**: 0.02 s (Plonsey is passive — no ODE integration)
- Output: `phie.igb` (extracellular potential time series), `phie_i.igb` (intracellular), `vm.igb` (transmembrane)

The dogbone pattern is visible in `phie.igb` at t ≈ 8 ms (end of stimulus): two lobes of opposite polarity positioned diagonally to the fiber axis, separated by a hyperpolarization zone along the fiber direction. This matches the expected result from Figure 02_16_dogbone_results.png in the tutorial.

## Lessons (transferable)

1. **Bidomain is required**: the dogbone pattern is a bidomain-only phenomenon. The intracellular and extracellular domains have different anisotropy tensors; their coupling produces the virtual electrode effect. Monodomain with a single effective tensor cannot replicate this.
2. **Unequal anisotropy condition**: dogbone requires g_il/g_it ≠ g_el/g_et. Standard cardiac values satisfy this (intra ratio ~9, extra ratio ~2.6). Setting all conductivities equal makes it elliptic.
3. **Plonsey (passive) model**: useful when you only care about the electric field, not the AP. No ions, no ODE. Reduces ionics compute to near-zero even for a 3D mesh.
4. **`crct.type=1`** = extracellular stimulus current injection. `crct.type=3` = ground (Dirichlet φ_e=0). For dogbone, both electrodes are extracellular only.
5. **Bath mesh**: the bath (tag 0, extracellular domain only) must be explicitly included. `phys_region[1]` covers both bath (tag 0) and myocardium (tag 1) as the extracellular domain. Bath conductivity is isotropic.
6. **Compute cost**: at 0.5 mm resolution, ~255 s single-core. At 0.25 mm it would be ~8× slower (volume ×8 for 3D). Use `--res 0.5` for exploration; the qualitative pattern is visible at this resolution.
