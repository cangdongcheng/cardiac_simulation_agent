# CLAUDE.md — 01_EP_single_cell / 01_basic_bench

## What this tutorial demonstrates

The basic single-cell EP workflow with `bench`:
1. Pace an isolated myocyte at fixed BCL with a suprathreshold stimulus current.
2. Observe approach to a stable limit cycle as pacing duration is extended.
3. Modify ionic-model parameters (e.g. peri-infarct zone reductions) via `--imp-par`.
4. Save state vector at end-of-pacing for reuse with `--read-ini-file`.

## Setup

| | value | source |
| --- | --- | --- |
| Solver | `bench` (single cell) | `tools.standard_parser` + `job.bench` |
| Default ionic model | `tenTusscherPanfilov` | `--EP` (also: `DrouhardRoberge`, `OHara`) |
| Stimulus | `--stim-curr 30.0 µA/cm²` | run.py:521 |
| BCL | 500 ms (default) | `--bcl` |
| Duration | 500 ms default; 5000 ms (exp01/03) or 20000 ms (exp02/04) | `--duration` |
| numstim | `int(duration/bcl)+1` | run.py:522 |
| dt | 10 µs | run.py:526 |
| dt-out | 0.1 ms | run.py:533 |
| Output format | `--bin -v` → split per-state-variable `.bin` files (8-byte doubles) + Trace_0.dat | run.py:551 |

## Output files (binary mode `--bin`)

For `<EP>=tenTusscherPanfilov`, inside the job-ID directory:

| File | Content |
| --- | --- |
| `<EP>.t.bin` | Time vector (8 bytes/sample) |
| `<EP>.Vm.bin` | Membrane voltage (8 bytes/sample) |
| `<EP>.Iion.bin` | Total ionic current |
| `<EP>_<EP>.<sv>.bin` | Per-state-variable trace (Cai, CaSR, CaSS, M, H, J, …) |
| `<EP>_trace_header.txt` | Column names for `Trace_0.dat` (currents) |
| `<EP>_header.txt` | Header for state-variable bins (used by `sv2h5b`) |
| `Trace_0.dat` | ASCII matrix of currents (ICaL, IK1, IKr, IKs, INa, INaCa, INaK, IbCa, IbNa, IpCa, IpK, Ito, V) |
| `<expID>_<EP>_bcl_<bcl>_ms_dur_<dur>_ms.sv` | End-of-run state vector for use with `--read-ini-file` |

Reading bins in Python: `np.fromfile(path, dtype=np.float64)`. Each `.bin` has `duration/dt-out + 1` samples.

## Findings: Run results

### exp01 — TTP baseline, 5 s pacing (pre-existing reference)

`./run.py --EP tenTusscherPanfilov --duration 5000 --bcl 500 --ID exp01`

- 50,001 samples (dt-out=0.1 ms × 5000 ms +1).
- Resting Vm = −86.20 mV; peak ≈ 39.4 mV.
- APD90 per beat (10 beats): 304, 273, 280, 280, 280, 281, 281, 281, 282, 282 ms.
- Beat-to-beat drift after the first beat is small but visible (steady drift toward higher APD); model is **not yet at limit cycle**.

### exp02 — TTP baseline, 20 s pacing (pre-existing reference)

`./run.py --EP tenTusscherPanfilov --duration 20000 --bcl 500 --ID exp02`

- 200,001 samples; 40 beats.
- APD90 first → last: 304 → 289 ms (gradual settling).
- End-of-run Vm = −85.09 mV (saved in `.sv`); Cai = 0.135 µM, Nai = 7.97 mM, Ki = 137.3 mM.
- The saved `exp02_tenTusscherPanfilov_bcl_500_ms_dur_20000_ms.sv` is the limit-cycle initial vector; downstream tutorials reference this.

### exp03 — Peri-infarct, 5 s pacing (run 2026-04-28)

```
./run.py --EP tenTusscherPanfilov --duration 5000 --bcl 500 \
  --ID 2026-04-28_exp03_periinfarct \
  --EP-par "GNa-62%,GCaL-69%,GKr-70%,GK1-80%" \
  --overwrite-behaviour overwrite
```

`bench` prints applied modifications:
```
GNa  modifier: -62%   value: 5.63844
GCaL modifier: -69%   value: 1.2338e-05
GKr  modifier: -70%   value: 0.0459
GK1  modifier: -80%   value: 1.081
```

- Peak Vm collapses from ~39 → **20.1 mV** (GNa drop dominates upstroke).
- APD90 last 4 beats: 302.8, 303.3, 303.8, 304.3 ms — short pacing run shows only mild drift.
- |odd−even| APD ≈ 4.6 ms — alternans not yet visible.

### exp04 — Peri-infarct, 20 s pacing (run 2026-04-28)

```
./run.py --EP tenTusscherPanfilov --duration 20000 --bcl 500 \
  --ID 2026-04-28_exp04_periinfarct \
  --EP-par "GNa-62%,GCaL-69%,GKr-70%,GK1-80%" \
  --overwrite-behaviour overwrite
```

- 40 beats; wall time ~5.7 s (real-time factor ≈ 12).
- APD90 last 4 beats: **314.8, 337.1, 315.0, 337.3 ms** → clear period-2 **APD alternans** (~22 ms swing).
- |odd−even| APD over last 10 beats: 17.7 ms (vs 4.6 ms in exp03 short run).
- Cai peak collapses from 0.535 µM (early) → 0.155 µM (late) — calcium-cycling instability characteristic of peri-infarct phenotype.
- Confirms tutorial's stated finding: "Calcium-cycling in the peri-infarct myocyte shows alternans".

## Lessons (transferable)

1. **`--imp-par` syntax** — comma-separated `param[+|−|*|/|=][+|−]N[%]` modifiers. `bench` echoes them to stdout at startup; always check this line to confirm parsing.
2. **Limit cycle requires 20+ s of pacing** for tenTusscherPanfilov at BCL=500. 5 s of pacing leaves substantial drift (especially in slow K+ gates and Cai handling).
3. **Saved `.sv` file** is the bridge between experiments. Naming convention: `<expID>_<EP>_bcl_<bcl>_ms_dur_<dur>_ms.sv`. Pass to next run with `--init=<file.sv>`.
4. **Binary output (`--bin`)**: separate per-variable `.bin` files of float64; load in numpy via `np.fromfile`. ASCII alternative drops `--bin` and writes everything to `<EP>.dat`.
5. **Existing exp01/exp02 dirs** in this tutorial are pre-shipped reference runs; `--overwrite-behaviour overwrite` is the cleanest way to re-run, or use a date-prefixed `--ID` to keep both.
6. **Peri-infarct phenotype** is reproduced with the documented `GNa-62%,GCaL-69%,GKr-70%,GK1-80%` recipe and is cleanly visible after 20 s of pacing at BCL=500: peak Vm collapse + APD alternans + Cai amplitude reduction.

## Carputils integration notes

- `@tools.carpexample(parser, jobID, clean_pattern='exp*')` — the `clean_pattern` is what `--clean` removes between runs.
- `job.bench(cmd, msg=...)` runs `bench` with the assembled flag list. cmd does NOT include the `bench` executable name itself; carputils adds it.
- After bench finishes, run.py manually moves all `*.txt`, `*.dat`, `*.sv` from the cwd into `job.ID/` because bench doesn't have a "place output in subdir" mode.
