# CLAUDE.md — 01_EP_single_cell / 12_parallel_single_cell

## What this tutorial demonstrates

**Population of Models (PoMs)** workflow on a single-cell ionic model. The `__main__` block in `run.py` builds a 200-sample Latin Hypercube across 10 parameters of the **Severi sinoatrial node** model, then dispatches each sample to a separate process via `multiprocessing.Pool`. Each subprocess invokes `bench` with the assembled `--imp-par` parameter string.

This is the closest the carputils-shipped tutorials come to demonstrating a "calibration" workflow for atrial/SAN cells.

## Setup

| | value | source |
| --- | --- | --- |
| Solver | `bench --imp Severi` | run.py:173, 227 |
| Sampling method | `scipy.stats.qmc.LatinHypercube(d=10)` → 200 samples scaled to [0.5, 2.0] | run.py:237-241 |
| Duration | 250000 ms (250 sec) | run.py:228 |
| Output window | last 5 sec only (`--start-out 245000`) | run.py:229 |
| Stim current | 0 (Severi auto-paces, no stim needed) | run.py:230 |
| Multipliers | `P_CaL P_CaT K_NaCa INaK_max Gf_Na_max Gf_K_max GKs_max GKr_max Gto_max GNa_max` | run.py:245 |
| Pool size | `multiprocessing.Pool()` (default = CPU count) | run.py:266 |

## Output

Each sample produces `Trace_<N>.dat` (currents trace) and a renamed `<imp>_openCARP_<dur>ms_dt<dtout>ms_<param_string>.dat` file in cwd via `os.replace` at run.py:218.

**No automatic subdir per sample** — the script intentionally uses `mkdir=False` in the decorator (run.py:162) so all 200 outputs land in the same directory, distinguished by filename.

## Findings: Run results (2026-04-29)

I built a smaller demonstrator at `run_small.py` (5 samples, 10 sec each, seed=42 for reproducibility, `--bin` output to `lhs_traces/`). Key result:

| sample | Vm_min (mV) | Vm_max (mV) | #APs in 1 sec | Period (ms) | HR (bpm) |
|---|---|---|---|---|---|
| 0 | −58.0 | +5.4 | 2 | 1000 | 60 |
| 1 | −65.3 | +19.7 | 2 | 1000 | 60 |
| 2 | −53.2 | +26.5 | 3 | 500 | 120 |
| 3 | −62.4 | +23.9 | 3 | 500 | 120 |
| 4 | −58.6 | +29.3 | 3 | 500 | 120 |

- **Heart rate variability** across the LHS samples: 60 ↔ 120 bpm. This is the kind of output PoMs are built to reveal.
- **Vm_max varies 5 → 29 mV** — some parameter combinations produce weakened APs (sample 0 has Vm_max only +5 mV), consistent with a depolarized resting potential.
- **Real-time factor ≈ 3.5** for Severi at dt=0.01 ms. So one 250-sec sim takes ~70 sec; 200 samples ≈ 4 hours sequentially, ~30 min on 8 cores.

## Lessons (transferable)

1. **The `__main__` block is the workflow** — the decorated `run(args, job)` function does the bench invocation, but the LHS sampling and the multiprocessing dispatch live in `if __name__ == '__main__':`. So you must run with `python3 run.py`, not as an imported module.
2. **`carpexample` decorator with `mkdir=False`** allows multiple "runs" to share a single output directory. Crucial when sweeping hundreds of parameter combinations — otherwise you get hundreds of subdirs.
3. **`bench --imp-par`** accepts a comma-separated string of `name*value` (multiplier) or `name=value` (absolute) modifiers. The script builds the string by iterating over `param_names` × `sample_value` pairs.
4. **Pool size** — `multiprocessing.Pool()` with no args uses `os.cpu_count()`, which on this machine is 8. Override with `Pool(processes=N)` for explicit control. The tutorial recommends matching to physical core count (not hyperthreads) — disk and memory pressure are real with hundreds of concurrent sims.
5. **Severi auto-paces** — sinoatrial node cells fire spontaneously, so `--stim-curr 0` is required. Adding a stimulus would cause an extra premature beat at t=0.
6. **Choice of parameters** is model-specific — Severi exposes `P_CaL`, `P_CaT`, etc. (verify with `bench --imp-info Severi`). For different ionic models, the `param_names` list at run.py:245 must be edited to match.
7. **LHS seed** is not set in the tutorial's `__main__` — every invocation produces a different sample. For reproducibility, pass `seed=...` to `LatinHypercube` (as `run_small.py` does for the demo).
8. **Output shape mismatch**: the tutorial renames `Trace_<N>.dat` (the per-sample currents trace from bench) to `<imp>_openCARP_..._.dat`. State variables are NOT captured in this default workflow — only the currents. To capture Vm/state vars, add `--bin --fout=...` to the bench cmd.

## Bug observation

Tutorial uses 250 sec / sample but only outputs the last 5 sec. The first 245 sec are computed and discarded — that's intentional (let Severi reach steady state) but wasteful if you only need pacing rate. For autopaced cells, ~10-20 sec of pacing is usually enough to identify the period; for human ventricular models with Ca cycling, 60+ sec may be needed.
