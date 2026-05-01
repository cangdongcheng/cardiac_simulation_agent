# CLAUDE.md — 02_EP_tissue / 05D_Region_Reunification

## What this tutorial demonstrates

The **`gvec[]` interface** for outputting state variables that span multiple ionic models. When a tissue has four regions using four different ionic models, a state variable like the sodium channel activation gate `m` may have different names in each model (e.g., `m/m/M/m` for LuoRudy94/Ramirez/tenTusscherPanfilov/MahajanShiferaw). The gvec interface creates a unified output field by specifying which per-model state variable maps to each output IGB file.

No experiment-specific CLI arguments. Running produces three IGB files: `INa_m.igb`, `INa_h.igb`, `INa_j.igb`.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | `TestSlabMesher2D_i.*` (same mesh as 05B/05C) | shipped |
| Conductivity | uniform: g_il=1.0, g_it=0.2, g_in=0.2 S/m | par file |
| tend | 40 ms | par file |
| dt / spacedt | 20 µs / 1 ms | par file |
| Stim | `LeftSideStimWithGaps.vtx` (point stim per region) | par file |
| LATs | threshold = −75 mV | par file |

### Ionic models per region

| Region tag | Model | INa gates |
| --- | --- | --- |
| 0 | LuoRudy94 | m, h, j |
| 1 | Ramirez | m, h, j |
| 2 | tenTusscherPanfilov | **M, H, J** (uppercase!) |
| 3 | MahajanShiferaw | m, h, j |

### gvec configuration (`Region-Reunification.par`)

```
num_gvecs = 3
gvec[0].name = INa_m
gvec[0].ID[0] = m   (LuoRudy94)
gvec[0].ID[1] = m   (Ramirez)
gvec[0].ID[2] = M   (tenTusscherPanfilov — note uppercase)
gvec[0].ID[3] = m   (MahajanShiferaw)
```

The `gvec[N].ID[]` array has one entry per `imp_region`. Array length is implicit (= `num_imp_regions`); no separate `num_ID` field.

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/05D_Region_Reunification
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py
```

## Findings (run on 2026-04-28)

Run completed in ~17 s (electrics 6 s, ionics 10 s).

Output directory: `2026-04-28/`. Files produced:
- `INa_m.igb`, `INa_h.igb`, `INa_j.igb` — gvec output (time series of each INa gate across all 4 regions)
- `init_acts_vm_act-thresh.dat` — LAT at −75 mV
- `vm.igb`

The gvec files provide a continuous spatial map of the activation/inactivation gate dynamics throughout the tissue even when four different models use different state variable names. This is the primary use case: visualization or analysis of a model-heterogeneous tissue where you want to compare a biophysical variable without per-model lookups.

## Lessons (transferable)

1. **`num_gvecs`** sets how many unified output files are produced. The `.ID[]` array for each gvec has length = `num_imp_regions` (not `num_gvecs`). No separate `.num_ID` field — unlike `gregion[].num_IDs`.
2. **tenTusscherPanfilov uses uppercase `M/H/J`** for the fast sodium gate variables. All other common models (`LuoRudy94`, `Ramirez`, `MahajanShiferaw`) use `m/h/j`. Always check `bench --imp-info <model>` for the exact variable names before building a gvec.
3. **The `WithGaps` stim file** (`LeftSideStimWithGaps.vtx`) places one stimulus electrode per region rather than a continuous left-edge plane. This is necessary here since each region has its own ionic model and may need an independently chosen stim node.
4. **gvec vs. phie_i / vm**: gvec outputs ionic model state variables, not electrical potentials. Suitable for gating variables (m/h/j), ion concentrations (Ca_i, Na_i), or custom ionic model parameters.
5. **The `Region-Reunification-WithPlugin-DoesNotWork.par`** in this directory demonstrates that the gvec approach does NOT work with ionic model plugins (only built-in IMP implementations). The file is a stub showing the limitation.
