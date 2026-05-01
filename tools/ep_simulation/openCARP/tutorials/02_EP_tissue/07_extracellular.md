# CLAUDE.md — 02_EP_tissue / 07_extracellular

## What this tutorial demonstrates

Three methods for computing extracellular potentials in openCARP:

1. **φ_e recovery (monodomain)**: efficient for a small number of field points. The cardiac source is assumed to be immersed in an unbounded homogeneous medium of conductivity σ_b. Computes potentials only at specified recovery points — not on the full domain.
2. **pseudo-bidomain**: extracellular potentials solved at output granularity only (no feedback from extracellular current upon excitation). Marginally more expensive than monodomain. Good trade-off for ECG studies.
3. **full bidomain**: full coupling; most accurate; requires explicit bath mesh and solves elliptic PDE every timestep (~10× more expensive than monodomain).

The demo uses a 3D transmural cardiac wedge (~10 mm transmural, one element thick circumferentially) with transmural+apico-basal heterogeneities from Boukens et al.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 3D wedge (transmural); `meshes/` dir | shipped |
| Resolution | 250 µm (default) | `--resolution` |
| Ionic model | Multi-region heterogeneous ventricular (built-in par file) | par file |
| Stim location | `--tm-stim-location {endo,epi}`, `--ab-stim-location {apex,mid,base}` | CLI |
| Duration | 60 ms (depol only) or 350 ms (full AP with T-wave) | `--duration` |
| Bath | 0.0 mm (default) | `--bath` |
| Source model | `--sourceModel {monodomain,bidomain,pseudo_bidomain}` | CLI |
| Field points | `ecg.pts` — recovery sites along apico-basal centerline | shipped |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/07_extracellular
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py \
  --sourceModel monodomain --duration 60

# pseudo_bidomain: adds phie.igb + phie_i.igb alongside phie_recovery.igb
python run.py --sourceModel pseudo_bidomain --ab-stim-location apex --duration 60

# Full bidomain with bath (longer compute)
python run.py --sourceModel bidomain --bath 5 --duration 60
```

## Outputs

| Source model | Files produced | Notes |
| --- | --- | --- |
| `monodomain` | `vm.igb`, `phie_recovery.igb`, `LATs` | only recovery points |
| `pseudo_bidomain` | `vm.igb`, `phie.igb`, `phie_i.igb`, `phie_recovery.igb` | full domain + recovery |
| `bidomain` | same as pseudo_bidomain | gold standard but slow |

`phie_recovery.igb` holds time-series potentials at each field point in `ecg.pts`.

## Findings (run on 2026-04-28)

Two runs completed:
1. `monodomain` — mid endo stim, 60 ms: `phie_recovery.igb` produced. Fast (~15 s).
2. `pseudo_bidomain` — apex endo stim, 60 ms: `phie.igb`, `phie_i.igb`, `phie_recovery.igb` all produced.

Key observation: in the monodomain case, only recovery field points have extracellular potential. In pseudo_bidomain the full domain φ_e is available for visualization but computed only at output time steps (no back-coupling).

## Lessons (transferable)

1. **`-phie_rec_ptf`** specifies the path to a `.pts` file with field point coordinates. Any number of field points can be listed; potentials at each are written as columns in `phie_recovery.igb`.
2. **Bath matters for accuracy**: with `--bath 0` (no bath), the recovery technique assumes an unbounded medium — recovered φ_e overestimates what a bidomain bath simulation gives. As bath size grows, the two converge.
3. **pseudo_bidomain vs. bidomain**: pseudo_bidomain is the practical choice for ECG comparisons — it gives the full φ_e field at negligible additional cost over monodomain, while bidomain ~10× slower.
4. **jobID encodes all parameters**: directory name pattern `<date>_ecg_<res>_um_<sourceModel>_tm_<endo/epi>_ab_<loc>_dur_<dur>_ms` makes it easy to identify runs.
5. **Stim delivery**: `stim[N].crct.type 0` = transmembrane current. The stimulus box is a thin 1.25 mm × 1.25 mm rectangle at the specified endo/epi / apex/mid/base location.
