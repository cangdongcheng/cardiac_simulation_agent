# CLAUDE.md — 02_EP_tissue / 03D_conduction_velocity_restitution

## What this tutorial demonstrates

How to compute **conduction velocity (CV) restitution** curves: CV as a function of S2 coupling interval (CI), quantifying how fast the tissue conducts a premature beat. CV decreases as CI shortens (less recovered tissue), eventually reaching a minimum CI below which conduction blocks entirely.

Tool used: `restituteCV` (carputils script at `settings.execs.RESTITUTECV`). It:
1. Paces `bench` to limit-cycle, then saves `.sv` state files at each CI
2. Creates a 1D cable, seeds each node with the CI-offset state, and fires S2
3. Measures CV at 0.25 and 0.75 cm, outputs `CVrestitution_<model>_bcl_<BCL>_beats_<N>.dat`

## Setup

| | value | source |
| --- | --- | --- |
| Cable mesh | 1.0 cm, 100 µm resolution | `restituteCV` defaults |
| Ionic model | `tenTusscherPanfilov` | `run.py:190` |
| Default Gil / Gel | 0.365 / 1.3111 S/m | `run.py:131-134` |
| S1 BCL | `--CI1` (500 ms default) | `run.py:195` |
| S1 beats | `--nbeats` (5) | `run.py:194` |
| CI range | `--CI0` to `--CI1` in steps of `--CIinc` | parser |
| Output | `CVrestitution_tenTusscherPanfilov_bcl_500_beats_5.dat` | `run.py:204` |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/03D_conduction_velocity_restitution
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py \
  --CI0 275 --CI1 500 --CIinc 25 --nbeats 5
```

Other tasks from docstring:
```bash
# Task 2: Gil reduced 75%
./run.py --CI0 275 --Gil 0.09125 --nbeats 5
# Task 3: Gil increased 50%
./run.py --CI0 275 --Gil 0.5475 --nbeats 5
```

## Findings (run on 2026-04-28)

All three runs completed successfully after fixing `restituteCV` bug (see below). np=1, each run ~30–60 s.

### CV restitution table (tenTusscherPanfilov, BCL=500)

| CI (ms) | Default Gil=0.365 | Gil=0.09125 (−75%) | Gil=0.5475 (+50%) |
| --- | --- | --- | --- |
| 275 | 0.0000 (block) | 0.0000 (block) | 0.0000 (block) |
| 300 | 0.0000 (block) | **0.3169** | 0.0000 (block) |
| 325 | **0.6707** | 0.3834 | 0.0000 (block) |
| 350 | 0.7608 | 0.4274 | 0.0000 (block) |
| 375 | 0.8147 | 0.4549 | **0.9470** |
| 400 | 0.8493 | 0.4729 | 0.9861 |
| 450 | 0.8881 | 0.4934 | 1.0312 |
| 500 (S1) | 0.9067 | 0.5033 | 1.0529 |

### Key results

| parameter set | minimum CI | CV at S1 BCL (m/s) |
| --- | --- | --- |
| Default (Gil=0.365) | **325 ms** | 0.907 |
| Gil=0.09125 (−75%) | **300 ms** | 0.503 |
| Gil=0.5475 (+50%) | **375 ms** | 1.053 |

Matches tutorial docstring task solutions exactly (min CI 325/300/375 ms).

### Interpretation

- **Lower conductivity → slower CV → lower minimum CI.** With Gil=0.09125 the absolute CV is ~halved; the minimum CI drops to 300 ms. This is due to reduced electrotonic coupling: less current diffuses ahead of the wavefront, so neighboring cells recover sooner in time (and space) relative to the stimulus.
- **Higher conductivity → faster CV → higher minimum CI.** With Gil=0.5475 the minimum CI rises to 375 ms. Higher electrotonic coupling slightly prolongs APD (source-sink loading during plateau), extending the effective refractory period.
- The restitution curves all flatten at long CIs, approaching the S1 steady-state CV. The shape of the restitution curve reflects the ionic model's recovery kinetics.

## Bug: `restituteCV` missing parser arguments

`restituteCV` calls `tuning.CVtuning.config(args)` which expects several arguments that its parser did not define. Fixed by adding to `restituteCV`'s `numopts` group:

```python
--beqm   (int, default=1)           # bidomain equiv monodomain
--stimV  (action='store_true')      # voltage clamp initiation
--mode   (choices=['gm','gi'])      # tune harmonic mean or gi
--surf   (bool, default=False)      # surface mesh
--pcl    (float nargs='+')          # pacing cycle length list
--log    (str, default='restituteCV.log')
--cvres  (str, default='cv_restitution.dat')
--table  (str, default='table.dat')
```

Fix is in `/usr/local/lib/opencarp/share/carputils/bin/restituteCV` (lines 102–120 after fix).

Note: the inner `restituteCV` job creates an intermediate dir `imp_<model>_vel_<v>_dx_<dx>/`. Between runs with the same model/velocity/resolution, this dir must be cleared or it prompts interactively (raw_input after MPI → exception). Clean it manually: `rm -rf imp_tenTusscherPanfilov_vel_0.6_dx_100.0/` between runs.

## Lessons (transferable)

1. **CV restitution minimum CI ≈ effective refractory period (ERP).** It's where conduction block first occurs; below it, S2 cannot propagate.
2. **Higher conductivity raises the minimum CI** via electrotonic APD prolongation (source-sink loading) — not a pure propagation-speed effect.
3. **`restituteCV` is a wrapper around `tuning.CVtuning`**, the same class used by `tuneCV`. It first paces `bench` to compute `.sv` limit-cycle states at each CI, then launches 1D cable sims to measure CV.
4. **Output format**: `CVrestitution_<model>_bcl_<BCL>_beats_<N>.dat` with 2-line header then `CI CV` columns. `CV=0` means conduction block (LAT at 0.75 cm was never recorded or was `inf`).
5. **The `.sv` state files are reused** between experiments at the same BCL/nbeats/model — the limit-cycle pacing step is skipped if `--svinit` is provided.
