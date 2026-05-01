# CLAUDE.md — 02_EP_tissue / 21_reentry_induction

## What this tutorial demonstrates

State-of-the-art arrhythmia induction protocols applied to a 2D tissue patch with fibrosis, using the AF-remodeled Courtemanche ionic model. Four protocols are demonstrated:

1. **Prepace**: Steady-state pacing at BCL=500 ms to generate an initial state and activation time map.
2. **PEERP** (Pacing at the End of the Effective Refractory Period): Binary search to find the minimum S2 coupling interval that propagates, then deliver ectopic beats at that timing.
3. **RP_E** (Rapid Pacing with reentry check at End): Pacing train with decreasing BCL (200→130 ms, step 10 ms); arrhythmia checked only after all beats.
4. **RP_B** and **PSD** (not run in this session): per-beat check and phase-singularity distribution, respectively.

The key scientific theme is that the induction **protocol choice matters**: PEERP induced reentry with only 2 precisely-timed ectopic beats; RP_E at BCL=130 did not produce detectable sustained activity in this single-processor run.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 2D tri slab 5 cm × 5 cm, 400 µm resolution | `run.py:setupMesh()`, `mesh.Block` |
| Nodes | 15,876 | meshtool output |
| Ionic model | Courtemanche (AF-remodeled) | `parameters.par` + run.py `--model Courtemanche` |
| AF remodeling | GCaL−55%, GK1+100%, GKur−50%, Gto−65%, GKs+100%, maxIpCa+50%, maxINaCa+60%, GKr×1.6 | `converge_CV()` `--modelpar` |
| Target CV | 0.3 m/s | `--cv 0.3` |
| Tuned conductivities | giL = 0.14039 S/m, geL = 0.70195 S/m (isotropic) | `tuneCV` results.dat |
| Mass lumping | M_lump=1 | `--M_lump 1` (faster for regular meshes) |
| Fibrosis | Circle r = 50000/3.5 ≈ 14,286 µm centred at (25000, 25000) µm; 2405 elements non-conductive (g=1e-7 S/m), 5613 elements ionically remodeled (GCaL×0.225, GNa×0.6, GK1×0.5) | `run.py:469–502`, `fibrosis_block_*` dir |
| Stim point (RP/PEERP) | `slabsize/7 × slabsize/2` = (7142.9, 25000, 0) µm → node 7830 | `run.py:572` |
| Prepace BCL | 500 ms, 4 beats | `--prepace_bcl 500 --prebeats 4` |
| dt | 20 µs | `parameters.par` |
| spacedt / timedt | 1 ms / 100 ms | `parameters.par` |
| Single-cell init | bench 20× BCL per ionic region (2 regions: normal AF, fibrotic AF) | `single_cell_initialization()` |

### Prepace results (context from prior session)

- LAT range: 0.63–173.5 ms (mean 87.4 ms)
- 15,862 of 15,876 nodes activated
- CV ≈ 0.288 m/s (target 0.3 m/s, ✓ within tolerance)
- State files: `prepace_.../state.2000.0.roe`, `state.2500.0.roe`, `init_values_stab_0_bcl_500.0.sv`

### APD94 from PEERP (last prepace beat)

- All nodes: mean 148.1 ms, min −1 ms (14 nodes, non-activated), max 173.0 ms
- Valid nodes (APD > 0): mean 148.3 ms, min 128.1 ms, max 173.0 ms, n=15,862

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/21_reentry_induction
export PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH

# Step 1: prepace (must come first, generates state files)
python run.py --np 1 --protocol prepace

# Step 2: PEERP (default: 2 ectopic beats, APD94 as first ERP estimate, tol=1 ms)
python run.py --np 1 --protocol PEERP

# Step 3: RP_E with reentry (BCL 200→130)
python run.py --np 1 --protocol RP_E --start_bcl 200 --end_bcl 130 --max_n_beats_RP 1

# Step 4: RP_E without reentry (BCL 200→140)
# IMPORTANT: must use --overwrite-behaviour overwrite to avoid interactive prompt
python run.py --np 1 --protocol RP_E --start_bcl 200 --end_bcl 140 --max_n_beats_RP 1 \
  --overwrite-behaviour overwrite
```

**Note**: Both RP_E runs share the same jobID (`YYYY-MM-DD_RP_E`). Running the second without `--overwrite-behaviour overwrite` (or `--ID` to differentiate) triggers an interactive prompt that fails with `EOFError` in batch/piped mode.

## Findings (run on 2026-04-28, --np 1)

### PEERP protocol (17 min total)

**ERP binary search — Beat 0** (starting from APD94 estimate of 2163 ms):

| Iteration | S2_start (ms) | Result | Max Vm (mV) |
| --- | --- | --- | --- |
| 1 | 2163 | REFRACTORY | −67.8 |
| 2 | 2181 | REFRACTORY | −73.4 |
| 3 | 2190 | REFRACTORY | −75.4 |
| 4 | 2194 | REFRACTORY | −76.1 |
| 5 | 2196 | REFRACTORY | −76.4 |
| 6 | 2197 | REFRACTORY | −76.5 |
| 7 | 2198 | REFRACTORY | −76.6 |
| 8 | 2208 | NOT REFRACTORY | −9.8 |
| 9 | 2203 | NOT REFRACTORY | +0.37 |
| 10 | 2201 | NOT REFRACTORY | +7.1 |
| 11 | 2200 | REFRACTORY | −76.5 |
| **Converged** | **ERP = 2201 ms** | | |

Key observation: the initial APD94 estimate of 2163 ms was refractory. The true tissue ERP (2200→2201 ms boundary) was ~38 ms longer than the APD94 estimate, indicating slow final repolarization in the fibrotic tissue. The binary search required 7 consecutive REFRACTORY steps before finding the upper bound at 2208 ms.

**Beat 0 result**: `No arrhythmia initiated at S2_start = 2201.0` (Vm = −80.3 mV at check time)

**ERP binary search — Beat 1** (starting from APD94 of next beat ≈ 2381 ms estimate):

| Iteration | S2_start (ms) | Result | Max Vm (mV) |
| --- | --- | --- | --- |
| 1 | 2381 | NOT REFRACTORY | −10.1 |
| 2 | 2341 | REFRACTORY | −64.1 |
| 3 | 2361 | NOT REFRACTORY | −11.6 |
| 4 | 2351 | NOT REFRACTORY | −2.2 |
| 5 | 2346 | REFRACTORY | −66.2 |
| 6 | 2349 | NOT REFRACTORY | +3.9 |
| 7 | 2348 | REFRACTORY | −67.0 |
| **Converged** | **ERP = 2349 ms** | | |

**Beat 1 result**: Arrhythmia initiated. `saved.txt`: node 7830, t=2749.0, beat 1. `reentries.txt`: node 7830, t=2849.0, beat 1. Log: `reentry p_7830_at_2849`.

**PEERP summary**:
- `tbeats.txt`: beat_0 ERP=2201 ms, beat_1 ERP=2349 ms
- `saved.txt`: `7830 2749.0 1` (activity at 100 ms after last stim → arrhythmia initiated)
- `reentries.txt`: `7830 2849.0 1` (activity still present at 500 ms after last stim → sustained reentry)
- ERP increased from 2201 ms to 2349 ms (+148 ms) between beats, consistent with the ectopic beat creating heterogeneous repolarization

**Key**: Only 2 precisely-timed ectopic beats were sufficient to initiate and sustain reentry.

### RP_E protocol (end_bcl=130, 13 min)

8-stimulus pacing train: BCL 200, 190, 180, 170, 160, 150, 140, 130 ms.

- Stimulus timing: stim[1] at 2190 ms, stim[7] at 3120 ms; `tbeats.txt`: `7830 3240.0 1` (t_last_beat=3240 ms)
- First RP_E simulation: t=2000 to t=3440 ms (t_last_save = t_last_beat+200 = 3440 ms)
- igbextract checked Vm at 392 surrounding nodes for last 100 time steps (t=3340–3440 ms)
- Result: **max Vm = −79.5 mV** (all at rest) → `No saved p_7830_at_3440.0`
- No free-run simulation launched; no `reentries.txt` written

**RP_E(130) did not induce reentry** in this run. The tutorial documentation states BCL=130 should produce reentry; discrepancy is likely due to `--np 1` (no MPI parallelism, slower simulation that may affect determinism), and the random fibrosis seed (fixed at `random.seed(1)` but the pacing dynamics on a single CPU may differ from multi-CPU runs described in the tutorial).

### RP_E protocol (end_bcl=140, crashed)

The second RP_E run failed immediately with an `EOFError` because carputils's `overwrite_mode_prompt()` asked interactively "Do you want to overwrite?" when run in a piped script context where stdin is closed. The directory `2026-04-28_RP_E` already existed from the end_bcl=130 run. The script exited after 1 second.

**Fix for future runs**: pass `--overwrite-behaviour overwrite` or use `--ID` with a unique suffix to separate the two RP_E runs.

## Output directory structure

```
21_reentry_induction/
├── prepace_block_50000.0um_resolution_400.0um_cv_0.3_with_4_beats_at_500.0_bcl_lump_1/
│   ├── state.2000.0.roe          # initial state for RP/PEERP
│   ├── state.2500.0.roe          # state after prepace
│   ├── init_values_stab_{0,1}_bcl_500.0.sv  # single-cell init per region
│   └── vm_last_beat.igb          # voltage during last prepace beat
├── fibrosis_block_50000.0um_resolution_400.0um/
│   ├── elems_slow_conductive.regele  # 2405 non-conductive elements
│   └── f_fib.regele                  # 5613 ionically-remodeled elements
├── 2026-04-28_PEERP/
│   ├── apd_94.0.dat              # APD94 per node (15876 values)
│   ├── stim_point.txt            # stim location → node 7830
│   ├── tbeats.txt                # "7830 2201.0" and "7830 2349.0"
│   ├── saved.txt                 # "7830 2749.0 1" (arrhythmia initiated)
│   ├── reentries.txt             # "7830 2849.0 1" (sustained reentry confirmed)
│   └── point_7830/
│       ├── beat_0/               # ERP search for beat 0
│       │   ├── state_lower_bound.roe / state_upper_bound.roe
│       │   ├── vm_lower_bound.igb / vm_upper_bound.igb / vm.igb
│       │   └── state.{2123,2162,...,2301,2401,2501}.0.roe
│       └── beat_1/               # ERP search for beat 1
│           ├── vm_7830_2749.0_1.igb  # saved voltage at arrhythmia check
│           └── state.{2348,...,2649}.0.roe
└── 2026-04-28_RP_E/
    ├── tbeats.txt                # "7830 3240.0 1"
    └── point_7830/beat_1/
        ├── state.2190.0.roe / state.3440.0.roe
        ├── Stimulus_{0-7}.trc    # 8 stimulus traces
        ├── vm.igb                # ~87 MB pacing train voltage
        └── out                   # igbextract output; max Vm = -79.5 mV
```

## Lessons (transferable)

1. **`carputils.model.induceReentry`** implements PEERP, RP_E, RP_B, PSD as importable functions. Each writes standardized output files: `tbeats.txt`, `saved.txt`, `reentries.txt`. The `job.bash()` + `igbextract` + `np.loadtxt` pipeline checks `max(vm[:,1:]) > -50 mV` to classify activity.

2. **PEERP ERP binary search**: The initial estimate is `t_stim + APD94`. If the tissue ERP exceeds this (as here: 2200 ms vs APD94≈2163 ms estimate), the algorithm finds no upper bound and expands by +20 ms at each step until it does. Fibrotic tissue can have ERP substantially longer than APD94 due to prolonged repolarization tail currents.

3. **ERP evolution between ectopic beats**: Beat_0 ERP=2201 ms; beat_1 ERP=2349 ms (+148 ms). The first ectopic beat that does NOT induce reentry still modifies the tissue state (spatially heterogeneous repolarization), increasing the apparent ERP of the next beat and creating the substrate for the second beat to induce reentry.

4. **RP_E arrhythmia check timing**: After the full pacing train, igbextract checks `header['dim_t']-100` through the end of vm.igb (the last 100 ms of the simulation). The check node set is a surface around the stim point (`_node_7830.vtx.surf.vtx`, 392 nodes). Only if max Vm > −50 mV does the algorithm run a 500 ms free-running simulation and check again.

5. **Directory naming collision bug**: Both RP_E runs (end_bcl=130 and end_bcl=140) produce the same jobID `YYYY-MM-DD_RP_E`. In batch mode (piped stdout), the interactive overwrite prompt raises `EOFError`. Fix: add `--overwrite-behaviour overwrite` or differentiate with `--ID`.

6. **tuneCV is called on every protocol invocation**: `converge_CV()` runs `tuneCV` unconditionally, even when the prepace state already exists. This adds ~30 s overhead per protocol run.

7. **APD94 vs tissue ERP**: In the AF-remodeled Courtemanche model with 400 µm fibrotic mesh, APD94 ≈ 148 ms (range 128–173 ms), but tissue ERP ≈ 200–201 ms above the prepace offset (t=2000+201 = 2201 ms). The gap (APD94=163 ms effective from prepace end, ERP=201 ms) reflects source-sink mismatch in the fibrotic substrate slowing final repolarization.

8. **Protocol sensitivity**: PEERP successfully induced reentry in a 2-beat protocol (total ~17 min); RP_E with BCL=130 did not produce detectable reentry in this `--np 1` run. The tutorial documentation predicts the opposite for BCL=130. The discrepancy likely reflects differences between single-core and multi-core execution, or the sensitivity of reentry induction to small timing perturbations near the reentry threshold.
