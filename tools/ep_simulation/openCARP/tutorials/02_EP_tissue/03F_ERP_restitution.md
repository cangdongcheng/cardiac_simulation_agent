# CLAUDE.md — 02_EP_tissue / 03F_ERP_restitution

## What this tutorial demonstrates

How to compute **effective refractory period (ERP) restitution** using the S1S2 bisection method: the ERP is the longest S1–S2 interval that fails to elicit a propagating response in tissue. Restitution describes how ERP shortens as the preceding diastolic interval (DI) decreases (faster pacing → shorter rest → shorter ERP).

The tutorial uses the **Courtemanche atrial model with AF remodeling** (ionic conductance scaling that mimics chronic AF-induced electrical remodeling).

## Setup

| | value | source |
| --- | --- | --- |
| Ionic model | `Courtemanche` + AF mods | `--model_par "GK1*2,GKs*2,GKr*1,GCaL*0.45,factorGKur*0.5,Gto*0.35,maxIpCa*1.5,maxINaCa*1.6"` |
| Mesh | 30×0.3×0 mm (2D surface, tri) | `run.py:335` |
| Resolution | dx=0.3 mm → 101×2=202 nodes | `--dx 0.3` |
| Target CV | 0.7 m/s | `--cv 0.7` |
| Pre-computed conductivities | giL=0.405, geL=1.45474 S/m | tutorial docstring |
| dt | 20 µs | `--dt 20` |
| Mass lumping | On (M_lump=1) | `--M_lump 1` |
| Single cell prepace | 100 beats at BCL=500ms | `--numstim 100 --cell_bcl 500` |
| Tissue prepace | 6 beats at each S1 | `--prebeats 6` |
| Stim current | 30 µA/cm² for 3ms | `--stim_strength 30 --stim_duration 3` |
| Sensing electrode | ¾ of mesh length from stim end | `run.py:746` |
| ERP tolerance | 1 ms | `--tolerance 1` |
| APD repolarization | APD94 | `--APD_percent 94` |

## AF remodeling conductance scaling

| Channel | Scale factor | Effect |
| --- | --- | --- |
| GK1 | ×2 | Larger IK1 → shorter plateau |
| GKs | ×2 | Larger IKs → faster repolarization |
| GKr | ×1 | Unchanged |
| GCaL | ×0.45 | Smaller ICaL → reduced plateau |
| factorGKur | ×0.5 | Smaller IKur |
| Gto | ×0.35 | Smaller Ito |
| maxIpCa | ×1.5 | Larger IpCa pump |
| maxINaCa | ×1.6 | Larger NCX current |

## ERP detection algorithm (bisection)

Starting from `[s2_min, s2_max] = [APD−10, ACT+APD]`, the algorithm bisects to find the longest S2 interval that fails to produce activation at the sensing electrode (`nodes_to_check`), using scipy `find_peaks` on Vm time series with threshold=10 mV/ms, capturing = 2+ peaks detected.

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/03F_ERP_restitution

# Single S1=500ms ERP calculation (skip tuneCV by providing conductivities)
MPLBACKEND=Agg PATH=/home/cdc/miniconda3/envs/opencarp/bin:/usr/local/lib/opencarp/lib/petsc/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py \
  --model Courtemanche --S1 500 --cv 0.7 --giL 0.40500 --geL 1.45474 \
  --model_par "GK1*2,GKs*2,GKr*1,GCaL*0.45,factorGKur*0.5,Gto*0.35,maxIpCa*1.5,maxINaCa*1.6" --np 1

# Full ERP restitution S1=500→200ms in steps of 50
MPLBACKEND=Agg PATH=... python run.py --model Courtemanche --S1 500 --restitution \
  --cv 0.7 --giL 0.40500 --geL 1.45474 \
  --model_par "GK1*2,GKs*2,GKr*1,GCaL*0.45,factorGKur*0.5,Gto*0.35,maxIpCa*1.5,maxINaCa*1.6" --np 1

# Custom S1 range: 600→100ms in 50ms steps
MPLBACKEND=Agg PATH=... python run.py --model Courtemanche --S1 600 --restitution 100 --S1_step 50 \
  --cv 0.7 --giL 0.40500 --geL 1.45474 \
  --model_par "GK1*2,GKs*2,GKr*1,GCaL*0.45,factorGKur*0.5,Gto*0.35,maxIpCa*1.5,maxINaCa*1.6" --np 1
```

## Findings (run on 2026-04-28)

### ERP restitution table (Courtemanche AF, CV=0.7 m/s, S1 500→200ms)

| S1 (ms) | ERP (ms) | DI (ms) | APD94 (ms) |
| --- | --- | --- | --- |
| 500 | 160 | 347 | 153 |
| 450 | 157 | 300 | 150 |
| 400 | 153 | 254 | 146 |
| 350 | 149 | 208 | 142 |
| 300 | 144 | 163 | 137 |
| 250 | 138 | 118 | 132 |
| 200 | 130 | 78 | 122 |

### Key observations

- **ERP decreases monotonically with shorter S1** (less recovery time → shorter refractory period)
- **ERP range: 130–160 ms** across S1=200–500ms — relatively flat restitution curve, characteristic of AF-remodeled atrial tissue
- **APD94 ≈ ERP − 7 ms** consistently across all S1 values (small gap; most of ERP is APD, not post-repolarization refractoriness)
- **DI = S1 − APD94**; at S1=200ms, DI=78ms (tissue barely recovering before next beat)
- The AF-remodeled Courtemanche model has shorter APD (122–153ms) than control atrial tissue (~200ms at BCL=500ms), reflecting AF-associated APD shortening

### Output files

- `results/2026-04-28_ERP_*/protocol.txt` — S1, ERP, DI, APD per row
- `results/2026-04-28_ERP_*/erp_vs_di.png` — ERP vs DI restitution curve plot
- `results/2026-04-28_ERP_*/erp_vs_s1.png` — ERP vs S1 plot
- `results/2026-04-28_ERP_*/capture.txt` — per-bisection capture/no-capture log

### State caching

The tutorial caches intermediate results aggressively:
- `init_state/single_cell/init_values_stab_numstim_100_bcl_<BCL>_stim_curr_30_imp_par_<mods>.sv` — single cell limit cycle
- `init_state/tissue/<mesh>_cv_<cv>_bcl_<bcl>/state.<time>.roe` — tissue prepace state

Second runs reuse these → only the S1S2 bisection steps run. Clean `init_state/` to force re-prepace.

## Key rules to remember

1. **Always provide `--giL`/`--geL`** to skip the `tuneCV` call inside the script; the values for Courtemanche AF at CV=0.7 m/s, dx=0.3 mm are giL=0.405, geL=1.45474.
2. **`MPLBACKEND=Agg`** prevents `plt.show()` from opening a GUI window — set this for all headless runs.
3. **Sensing electrode is at ¾ of mesh length** from the stimulus end, avoiding pacing artifacts. Nodes ~72–78 (itracel) + 173–179 (opposite face).
4. **ERP ≠ APD**: ERP ≥ APD by definition. The gap (post-repolarization refractoriness) is small (~7ms) in this model.
5. **`--np 1` is faster than `--np 4`** for this 202-node mesh — MPI overhead dominates.

## Lessons (transferable)

1. **Bisection is efficient for ERP**: ~7 iterations to achieve 1ms precision from a range of ~20ms. Linear scan would need 20 sims; bisection needs ~7.
2. **ERP restitution slope** indicates arrhythmia vulnerability: steep slope (ERP drops sharply at short DI) → higher re-entry risk. This AF model has a shallow slope, suggesting moderate vulnerability.
3. **State caching is essential** for parametric studies: tissue prepace is the slow step; bisection sims are fast. Cache the prepace state and reuse across S1 values.
4. **DI at the sensing node** is computed as S1−APD, not from the activation time directly. This can differ from the "true" DI if the wave arrival time at the sensing node differs from the pacing site.
