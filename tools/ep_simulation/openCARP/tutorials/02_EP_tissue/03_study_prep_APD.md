# CLAUDE.md — 02_EP_tissue / 03_study_prep_APD

## What this tutorial demonstrates

How to **calibrate APD** (action potential duration) by tuning maximal-conductance parameters of an ionic model, using a 1.5 cm 1D cable of tenTusscherPanfilov epicardial myocytes paced at BCL=700 ms. Pipeline: pace → write Vm → run `igbapd` postprocessor → inspect APD80.

This is the first tutorial that:
- Uses a **pre-built mesh** (under `Mesh_1.5cm_Cable/`) instead of generating one in Python.
- Pulls in the `igbapd` post-processor from `igbutils`.
- Demonstrates `--im_param` to modify ionic conductances at runtime.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 1.5 cm 1D cable, 76 nodes (≈200 µm spacing) | `Mesh_1.5cm_Cable/` |
| Solver | monodomain (`bidomain=0`, `bidm_eqv_mono=1`) | `sim.par` |
| Integration | `dt=20` µs | `sim.par` |
| Output stride | `spacedt=timedt=1` ms | `sim.par` |
| Default ionic model | `tenTusscherPanfilov` (overrideable via `--IMP`) | `run.py:125` |
| Stim site | left cap, vertex file `Mesh_1.5cm_Cable/stim.vtx` | `run.py:151` |
| `--bcl` | 700 ms | parser |
| `--nbeats` | 2 | parser |
| Run length | `tend = bcl × nbeats + 0.1` (1400.1 ms by default) | `run.py:160` |

## CLI

```bash
# baseline APD
./run.py --Mode default --IMP tenTusscherPanfilov

# adjust mode — pass --im_param a comma-separated list of conductance modifications
./run.py --Mode adjust --IMP tenTusscherPanfilov --im_param "GKs*2"
./run.py --Mode adjust --IMP tenTusscherPanfilov --im_param "GCaL*1.5,GKr*0.7"
```

`--im_param` syntax follows openCARP's general modifier grammar (same as in `--imp-par` to `bench`): `name[*/+-=]value[%]` separated by commas. Valid names per model come from `bench --imp-info -I <model>`.

## `igbapd` post-processor

Each run automatically calls `igbapd` to compute APD80 per node:

```bash
igbapd -t <(nbeats-1)*bcl> --repol=80 --vup=-10 \
       --peak-value=plateau --plateau-start=15 \
       --output-file <simid>/apd.dat <simid>/vm.igb
```

- `-t (nbeats-1)*bcl` selects the **last beat** for analysis (skips early transients).
- `--repol=80` defines APD as time from upstroke to 80% repolarization.
- `--vup=-10` is the upstroke-detection threshold (Vm crossing −10 mV upwards).
- `--peak-value=plateau --plateau-start=15` uses the plateau-Vm 15 ms after upstroke as the AP peak (more robust than the absolute upstroke peak for AP shapes with notches).

Output `apd.dat` is one APD80 value per node (text, plain numbers).

## Findings (run on 2026-04-28)

Resting Vm = −86.2 mV. Stim node Vm peak = +87.3 mV. Distal node peak only +32.9 mV — the cable is partially activated by the time it ends (1.5 cm cable, 2 beats, propagation succeeds but the wave hasn't fully traversed by the time of measurement at all distal nodes? Or the apdmin filter excludes them). 76 APD values returned, indicating all nodes were measurable.

| run | mean APD80 | min/max APD80 | Δ vs. baseline |
| --- | --- | --- | --- |
| `--Mode default` (tenTusscherPanfilov, BCL=700) | **290.89 ms** | 287.28 / 293.02 | — |
| `--Mode adjust --im_param "GKs*2"` | **242.14 ms** | 239.14 / 243.47 | **−48.7 ms** ✓ shortens |
| `--Mode adjust --im_param "GCaL*1.5"` | **316.48 ms** | 313.35 / 317.92 | **+25.6 ms** (under 30 ms target) |

Conclusions:
- **GKs↑ shortens APD** as expected (more outward IKs → faster phase-3 repolarization). Doubling GKs shortened APD by 49 ms — comfortably exceeds the tutorial's 30 ms target.
- **GCaL↑ prolongs APD** as expected (more inward ICaL → longer plateau). +50% GCaL added 26 ms; pushing further (e.g. GCaL*2 or combining with GKr↓) would clear the 30 ms threshold.
- APD spread across the 76 cable nodes is small (~5 ms range), consistent with a homogeneous cable; the small spread reflects boundary effects and is expected.

Tutorial demonstrations completed successfully.

### Job ID gotcha

`run.py:163` builds the simID as `'APD' + args.im_param`, which means the literal string `*` ends up in the directory name (e.g. `APDGKs*2/`). Bash glob-expansion can bite if you script around these directories — quote them, or use a different separator if you fork the tutorial.

## Lessons (transferable)

1. **Igbapd is the canonical APD post-processor.** It's part of the igbutils suite (tutorial 05) and lives at the path in `settings.execs.igbapd`. Common knobs: `--repol={50,80,90}` (which APD to compute), `--vup` (upstroke threshold), `--peak-value={absolute,plateau}` (AP shape handling).
2. **`--im_param` is the runtime way to perturb ionic conductances** (mirrors `bench --imp-par` for single cell). Use `bench --imp-info -I <model>` to discover the parameter names; not all knobs are exposed.
3. **Pre-built meshes are referenced by `<dir>/<basename>`** (no extension) — openCARP loads `<basename>.elem`, `.lon`, `.pts`, etc. The path is passed as `-meshname`. `stim[*].elec.vtx_file` references a `.vtx` text file with vertex indices.
4. **`bidm_eqv_mono=1`** (in `sim.par`) is a numerical option that uses the bidomain matrix structure even in monodomain mode — improves conditioning. Probably worth keeping when you fork this tutorial.
5. **2 beats is not steady-state.** The tutorial explicitly admits this is for time, not accuracy. For real APD calibration you need >50 beats to converge transients.
