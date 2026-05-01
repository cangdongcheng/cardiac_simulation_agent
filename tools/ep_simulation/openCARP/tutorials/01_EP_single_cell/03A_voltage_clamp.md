# CLAUDE.md — 01_EP_single_cell / 03A_voltage_clamp

## Status: documentation-only tutorial

The shipped `run.py` only has a docstring + parser stub:

```python
def parser():
    parser = tools.standard_parser(False)
    return parser
```

There is no `run()` body, so `./run.py` does nothing. Per the docstring, the tutorial is **explicitly marked incomplete** ("example incomplete!").

The tutorial demonstrates the underlying `bench --clamp` mechanism by direct command-line invocation, not via `carputils`.

## What the docstring shows

A voltage-clamp protocol on a tenTusscherPanfilov cell, then visualization in limpetGUI:

```bash
# 1. Run clamp experiment
bench --imp tenTusscherPanfilov --clamp -40.0 --clamp-dur 200 --clamp-start 10 \
      --validate --duration 500 --stim-curr 0. --fout=./VmClamp

# 2. Convert outputs to HDF5 for limpetGUI
bin2h5.py  VmClamp_header.txt                tenTusscherPanfilov_trace_header.txt  → svclamp.h5
txt2h5.py  tenTusscherPanfilov_trace_header.txt  Trace_0.dat                         → curclamp.h5

# 3. Visualize
limpetGUIpyQt svclamp.h5
limpetGUIpyQt curclamp.h5
```

(Note: `bin2h5.py`/`txt2h5.py` shown in the docstring; in modern openCARP the equivalent are `sv2h5b` and `sv2h5t` — both on PATH at `/usr/local/bin`.)

## Verified locally (2026-04-28)

Ran the docstring's bench command directly. It works without modification:

| t (ms) | Vm (mV) | Notes |
|---|---|---|
| 0   | −86.20 | Rest (TTP default initial state) |
| 10  | −86.04 | Just before clamp activation |
| 100 | −40.00 | During clamp (held at −40 mV) |
| 210 | −40.00 | Clamp release moment |
| 300 |  +26.75 | After release: cell enters spontaneous AP (residual ICaL activation, since stim_curr=0 but ionic currents still drive it) |
| 499 |  −43.01 | Late repolarization |

Currents at the clamp voltage of −40 mV (read from `Trace_0.dat` columns named in `tenTusscherPanfilov_trace_header.txt`):

- `IKr` peak during clamp: 0.016 µA/cm² @ t≈90 ms (slow activation toward steady-state at −40 mV)
- `IKr` tail current after release reaches 0.122 (the post-clamp depolarization opens the channel further)
- `ICaL` peak during clamp: −0.88 µA/cm² (inward, transient)

The tutorial figure (`/images/01_03_voltage_clamp.png`) shows IKr response — the recipe reproduces.

## Key bench flags for clamp experiments

| flag | purpose |
| --- | --- |
| `--clamp <V>` | Clamp Vm to specified value (mV) |
| `--clamp-dur <T>` | Duration of clamp (ms) |
| `--clamp-start <t>` | Time at which clamp turns on (ms) |
| `--stim-curr 0` | Disable current injection so only the clamp drives Vm |
| `--validate` | Validate model + write extra trace info |
| `--clamp-SVs <list>` | Clamp specific state variables (e.g. `Cai:Nai`); needs `--SV-clamp-files` |
| `--SV-clamp-files <files>` | Trace files providing the clamp values (one per SV) |
| `--AP-clamp-file <f>` | Drive Vm from an external AP trace (used for AP-clamp protocols) |
| `--clamp-ini` | Per-iteration sweep: holds at `--clamp-ini` voltage, then steps as listed in `--clamp-file` |
| `--clamp-file` | File of clamp voltages (one per line) for I-V curve sweeps |

## Lessons (transferable)

1. **Simple Vm clamp** = `--clamp <V> --clamp-dur <T> --clamp-start <t> --stim-curr 0`. Use `--validate` to write all current channels.
2. **State-variable clamps** (e.g. clamp Cai while letting Vm float) use `--clamp-SVs/--SV-clamp-files`. This is a powerful tool for dissecting which currents drive a phenomenon.
3. **Tutorial run.py is a stub** — no `@tools.carpexample` decoration, no `run()` body. To exercise the tutorial, run `bench` directly as in the docstring.
4. **Output naming**: `--fout=./VmClamp` produces `VmClamp.<sv>.bin` for all state variables and `VmClamp_header.txt` listing the SV names. The `Trace_0.dat` ASCII matrix is column-keyed via the *separate* `<imp>_trace_header.txt`.
5. **The cell fires after clamp release** unless you also clamp Cai or other slow variables — the protocol holds Vm but lets ionic gating evolve, leaving the cell primed for an AP once the clamp lifts.
