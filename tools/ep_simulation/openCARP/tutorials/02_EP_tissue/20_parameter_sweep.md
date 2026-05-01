# CLAUDE.md — 02_EP_tissue / 20_parameter_sweep

## What this tutorial demonstrates

The **parameter sweep / polling file** workflow in carputils. A two-step process:

1. **Generate a polling file** — specifies all parameter combinations to test. Can use linear or Latin hypercube sampling (`--sampling-type {linear,lhs}`).
2. **Run the sweep** — pass the polling file; carputils runs one openCARP invocation per row, each in a separate subdirectory.

The demo sweeps two stimulus start times on a 1 cm strand with voltage electrodes at both caps.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 1 cm strand, auto-generated | run.py |
| Ionic model | TenTusscherPanfilov (inferred from `stimtest.par`) | par file |
| Stimuli | `stim[0]` left cap, `stim[1]` right cap, voltage type (`crct.type=2`), strength=1000 µA/cm², duration=2 ms | parameters.par |
| tend | 100 ms | par file |
| Output per run | `vm.igb`, `Left.trc`, `Right.trc`, `parameters.par` | |
| Polling file | `stimulus.poll` (5 rows, shipped) | |

## Polling file format

```
-stim[0].ptcl.start 20.000000 -stim[1].ptcl.start 50.000000
-stim[0].ptcl.start 30.000000 -stim[1].ptcl.start 55.000000
...
```

Each line is a set of openCARP CLI flags that override the base command. The polling file used here was generated with 5 points (not 25 as shown in the docstring) across ranges [20–60] and [50–70] ms.

## Two-step workflow

### Step 1 — Generate polling file

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/20_parameter_sweep
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py \
    --polling-file stimulus.poll \
    --polling-param "stim[0].ptcl.start" "stim[1].ptcl.start" \
    --polling-range 20:60:5 50:70:5
```

This writes `stimulus.poll` with 5 parameter combinations.

### Step 2 — Run sweep

```bash
python run.py --polling-file stimulus.poll --np 1
```

Each combination runs in `<jobID>/p_stim0ptclstart.<v0>_stim1ptclstart.<v1>/`.

## Findings (run on 2026-04-28)

Run directory: `2026-04-28_np1/`. 5 parameter sets executed sequentially (~11 s each).

| stim[0].start (ms) | stim[1].start (ms) | Subdir |
| --- | --- | --- |
| 20 | 50 | `p_stim0ptclstart.20.000000_stim1ptclstart.50.000000` |
| 30 | 55 | `p_stim0ptclstart.30.000000_stim1ptclstart.55.000000` |
| 40 | 60 | `p_stim0ptclstart.40.000000_stim1ptclstart.60.000000` |
| 50 | 65 | `p_stim0ptclstart.50.000000_stim1ptclstart.65.000000` |
| 60 | 70 | `p_stim0ptclstart.60.000000_stim1ptclstart.70.000000` |

Output traces `Left.trc` and `Right.trc` have 401 time points (0–100 ms at 0.25 ms resolution). Both traces start identical because both electrodes apply the same voltage source.

## Lessons (transferable)

1. **`--polling-file` alone runs the sweep**; `--polling-file + --polling-param + --polling-range` generates the file. Never combine generation and execution flags in one call — the run step detects the polling file via `settings.cli.polling_file` and errors with `TypeError` if no file exists at execution time.
2. **Polling parameters must be set somewhere** (base `.par` file or run.py command assembly). carputils validates that each `--polling-param` name is actually present in the assembled command. If you sweep a parameter that's never set, it throws an error.
3. **Subdirectory naming is automatic**: `p_<param1name>.<val1>_<param2name>.<val2>/` — the directory name encodes all parameter values. Dots in parameter names are replaced by underscores in the dir name.
4. **Parallel sweeps**: `--np 8 --np-job 2` runs 4 simulations simultaneously each on 2 cores. `--np` must be a multiple of `--np-job`. Not usable on this system (no mpiexec) — use `--np 1` with no `--np-job`.
5. **`sampling-type lhs`** requires `pyDOE` package (not installed by default). Linear sampling always works.
6. **Custom polling files**: hand-write a file in the same format for non-uniform or non-linear parameter grids. carputils accepts any polling file as long as each line is a valid set of openCARP CLI flags.
7. **`polling_subdirs=False`** in `job.carp()` or `--simID` in polling file disables auto-subdir creation — all runs go into the same directory (not recommended for sweeps, useful for single re-runs).
