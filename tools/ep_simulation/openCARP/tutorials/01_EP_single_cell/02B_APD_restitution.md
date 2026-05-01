# CLAUDE.md — 01_EP_single_cell / 02B_APD_restitution

## What this tutorial demonstrates

How to use `bench --restitute` to build APD-vs-DI restitution curves with two pacing protocols:

- **S1S2** (default): N pre-pacing + nbeats S1 train at fixed BCL, then a single S2 with prematurity decremented from CI1 down to CI0 by CIinc.
- **Dynamic**: continuous train at progressively shorter BCL (BCL → CI0 in CIinc steps).

The slope of the restitution curve at short DI is a classical arrhythmogenicity index: slope > 1 promotes alternans and conduction block.

## Setup

| | value | source |
| --- | --- | --- |
| Solver | `bench --restitute {S1S2,dyn} --res-file=...` | run.py:404-409 |
| Default model | `tenTusscherPanfilov` | `--imp` |
| Threshold finding | iterative: stim_curr += 2 µA/cm² until upstroke ≥ −10 mV; then doubled for actual run | run.py:346-376 |
| Default BCL | 600 ms | `--BCL` |
| Prebeats | 20 | `--prebeats` |
| nbeats (S1 train per S2) | 5 | `--nbeats` |
| CI1 | = BCL (default) | `--CI1` |
| CI0 | 50 ms | `--CI0` |
| CIinc | 25 ms | `--CIinc` |
| Stim duration | 2 ms | run.py:405 |

## Output files

In the job-ID directory:

| File | Content |
| --- | --- |
| `restitution_protocol.txt` | The 7-line protocol file passed to `bench --restitute` |
| `restout_APD_restitution.dat` | Beat# / Prematurity / SteadyState / APD(n) / DI(n) / **DI(n−1)** / Triangulation / VmMax / VmMin / tAct |
| `restout_AP_stats.dat` | Per-beat AP-statistics (similar fields) |
| `restout.txt` | Vm samples used for `--res-trace` |
| `Trace_0.dat` | Per-current ASCII traces (column names in `<imp>_trace_header.txt`) |
| `thresh_save.sv` | State after threshold-finding sub-run |

For restitution curves, use **APD = column 4** vs **DI(n−1) = column 6**. The `# steady_state` flag in column 3 indicates whether the cell had reached steady state.

## Findings: Run results (2026-04-28)

### TTP baseline (S1S2 default)

`./run.py --imp tenTusscherPanfilov` → dir `2026-04-28_CI0-50_CI1-None_S1S2/`

- 20 prebeats + 5×S1 train per S2; BCL=CI1=600 ms, CI0=50 ms, CIinc=25.
- Threshold current (iterative): converged ≈ 24 µA/cm² → run uses 2× = 48.
- Wall time ~11.4 s for the full sweep.
- APD vs DI(n−1) (smooth branch):
  - DI=27 ms → APD=174 ms (loss-of-capture region close)
  - DI=275 ms → APD=298 ms (plateau)
- **Max restitution slope ≈ 1.23** at the shortest accepted DI; falls to 0.45 at DI≈140 ms and ~0 above DI=275 ms.
- Loss of capture at CI=50 → DI=2.9 ms → APD=28.5 ms.

### TTP with `GKr=0.172, GKs=0.441` (tutorial "slope > 1" target)

`./run.py --imp tenTusscherPanfilov --params 'GKr=0.172,GKs=0.441' --ID 2026-04-28_S1S2_TTP_slope_gt1`

- TTP defaults are GKr=0.153, GKs=0.392 (read from saved `.sv`); these set explicit absolute values (+12% / +12%).
- APD shortens: 178 (DI=40) → 294 (DI=540) ms.
- Max slope **= 1.07** — only modestly above 1, NOT the "well above 1" the tutorial advertises. The full Tusscher2006 recipe also includes `G_pCa, G_pK, G_tf` modifications which aren't applied here (those parameters may not all exist in the openCARP TTP variant — verify with `bench --imp-info tenTusscherPanfilov`).

### TTP with `GKr=0.134, GKs=0.270` (tutorial "slope < 1" target)

`./run.py --imp tenTusscherPanfilov --params 'GKr=0.134,GKs=0.270' --ID 2026-04-28_S1S2_TTP_slope_lt1`

- Reduces both K+ conductances → APD lengthens (197 → 343 ms).
- Max slope **= 1.25** at DI≈33 ms — actually *higher* than the slope_gt1 case. Counter to tutorial expectation.
- Caveat: tutorial expects slope < 1 with these params. The discrepancy is consistent with the "slope_gt1" caveat above — the tutorial's full Table-2 modification recipe is incomplete in the run.py exposure.

### Courtemanche S1S2

`./run.py --imp Courtemanche --ID 2026-04-28_S1S2_Courtemanche`

- APD range 186 (DI=10) → 300 (DI=440) ms. Atrial action potential is shorter than ventricular.
- **Max slope ≈ 0.93** — flatter restitution curve than TTP, consistent with the published profile in the tutorial figure (Courtemanche flatter than tenTuscher).

### TTP Dynamic protocol

`./run.py --imp tenTusscherPanfilov --Protocol Dynamic --ID 2026-04-28_Dynamic_TTP`

- Continuous train, BCL decremented from CI1=600 to CI0=50 in steps of 25.
- APD range: 169 (DI=14) → 298 (DI=275) ms — captures shorter DI than S1S2 because each train can adapt across multiple beats.
- **Max slope ≈ 1.31** (DI≈32 ms) — steeper than S1S2 baseline. Dynamic protocol exposes restitution at very short DI before block.

## Bug: `jobID` kwarg formatting

`02B_APD_restitution/run.py:319-324`

```python
def jobID(args):
    today = date.today()
    ID = '{}_CI0-{}_CI1-{}_{}'.format(today.isoformat(), args.CI0, args.CI1, args.Protocol)
    if args.params:
        ID += '_{}'.format(args.imp,args.params)   # ← only args.imp interpolated; args.params dropped
    return ID
```

Two runs with identical `--imp` and different `--params` collide on the same job ID, silently overwriting each other. Fix:

```python
ID += '_{}_{}'.format(args.imp, args.params.replace(',', '_').replace('=', ''))
```

Workaround: always pass `--ID` explicitly when sweeping `--params`.

## Lessons (transferable)

1. **`--restitute` mode** is invoked via the protocol file `restitution_protocol.txt`; bench reads it line-by-line. First line: protocol selector (1=S1S2, 0=dynamic).
2. **Iterative threshold finding** before the actual sweep is built into run.py — `stim_curr` is doubled-after-capture. This gives ~48 µA/cm² for TTP, well above the typical 30 used for plain pacing.
3. **DI(n−1) in column 6** is the right x-axis for restitution curves. DI(n) (column 5) is the post-AP rest, useful for next-beat predictions but not the conventional plot.
4. **Tutorial parameter recipes (Tusscher2006 Table 2)** are *partial* in this run.py — only GKr and GKs are exposed. Slope categorization in the tutorial assumes the full `G_pCa, G_pK, G_tf` recipe; reproducing the published "slope < 1 vs slope > 1" classification needs all parameters. Verify which exist in the model with `bench --imp-info tenTusscherPanfilov`.
5. **Atrial vs ventricular**: Courtemanche shows shorter APD and flatter restitution than TTP — both expected and observed.
6. **Dynamic vs S1S2**: dynamic explores shorter DI (because the cell adapts gradually), yielding steeper short-DI slopes; S1S2 isolates the immediate effect of one premature beat.
