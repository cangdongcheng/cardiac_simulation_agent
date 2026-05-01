# CLAUDE.md — 02_EP_tissue / 02_stimulation

Notes for Claude Code working in this tutorial dir. Loaded automatically alongside the parent `tutorials/CLAUDE.md`.

## What this tutorial demonstrates

Six configurations for **extracellular** stimulation on a thin 1 cm cardiac strand, contrasting voltage vs. current sources, asymmetric vs. balanced fields, and bidomain vs. monodomain handling of extracellular drive. Membrane physics is identical across modes; only the electrode wiring (`stim[*].crct.*`) changes.

`run.py` exposes only two CLI flags:

| flag | choices | default |
| --- | --- | --- |
| `--stimulus` | `extra_V`, `extra_V_bal`, `extra_V_OL`, `extra_I`, `extra_I_bal`, `extra_I_mono` | `extra_V` |
| `--grounded` | `on`, `off` | `on` |

Everything else (mesh, ionic model, stim geometry, pulse timing) is hard-coded in `run.py` + `stimtest.par`.

## Setup at a glance

| | value | source |
| --- | --- | --- |
| Mesh | `mesh.Block(size=(10, 0.1, 0.1))` mm | `run.py:558` |
| Fibres | longitudinal (0°/0°/90°/90°) | `run.py:560` |
| Ionic model | `DrouhardRoberge`, `im_param="APDshorten*8"` | `stimtest.par` |
| Default coupling | bidomain (`bidomain=1`) | `stimtest.par` |
| Sim duration | `tend=5` ms | `stimtest.par` |
| Output stride | `spacedt=timedt=0.1` ms | `stimtest.par` |
| Stim pulse | starts at 2 ms, 2 ms wide, 1 pulse | `stimtest.par` |
| Tags | `IntraTags=[1]`, `ExtraTags=[1]` | `run.py:584-585` |

## Electrode geometry (µm)

Three electrodes are pre-positioned. `lstm` is electrode A (left), `rstm` is B (right), `astm` (when used) is the auxiliary C defined in `stimtest.par:24-32`.

| | x range | y range | z range |
| --- | --- | --- | --- |
| **A** (left cap) | [−5050, −4950] | [−100, 100] | [−100, 100] |
| **B** (right cap) | [4950, 5050] | [−100, 100] | [−100, 100] |
| **C** (centre, full y-z) | [−10, 10] | [−5000, 5000] | [−5000, 5000] |

A and B are box-electrodes covering the strand's caps; C is a thin slab cutting the entire strand at x=0.

## `stim[*].crct.type` codes used in this tutorial

| code | meaning |
| --- | --- |
| 0 | unset / transmembrane current (default in `stimtest.par` for stim[0] and stim[2]) |
| 1 | extracellular current source |
| 2 | extracellular voltage source |
| 3 | extracellular ground (Dirichlet 0) |
| 5 | extracellular voltage with switched / open-after-pulse circuit |

Look up additional codes via `carphelp stim+crct\.type --no-col` if you need them.

## Mode matrix

| mode | A (`stim[0]`) | B (`stim[1]`) | C (`stim[2]`) | bidomain |
| --- | --- | --- | --- | --- |
| `extra_V` | type=2, +2 kV | GND (type=3) | unused | 1 |
| `extra_V_bal` | type=2, +1 kV | type=2, −1 kV | GND (if `--grounded on`) | 1 |
| `extra_V_OL` | type=5, +2 kV (switched) | GND | unused | 1 |
| `extra_I` | type=1, +3.1e6 µA/cm³ | GND | unused | 1 |
| `extra_I_bal` | type=1, −3.1e6 µA/cm³ | type=1, `crct.balance=0` (auto-balance to A) | GND (if `--grounded on`) | 1 |
| `extra_I_mono` | type=1, +3.1e6 µA/cm³ | type=1, `crct.balance=0` | GND (if `--grounded on`) | 0, with `extracell_monodomain_stim=1` |

`--grounded off` semantics:
- `extra_V`, `extra_V_OL`: wipes `rstm` (B no longer grounded). Pure single-Dirichlet setup; mostly a theoretical case.
- `extra_V_bal`, `extra_I_bal`, `extra_I_mono`: drops the C ground. For balanced current cases this leaves a pure-Neumann problem; openCARP will auto-detect and use the nullspace constraint (mean-zero phie).

`crct.balance=0` on B in the `_bal` modes is **not** "no current" — it means "this stimulus's strength is computed automatically to balance stim[0]". Confirm this in the produced `parameters.par`.

## Running and verification recipe

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/02_stimulation
/home/cdc/miniconda3/envs/opencarp/bin/python run.py --stimulus <mode>
```

Each run is cheap: ~1 cm strand, 5 ms, np=1, finishes in seconds.

After a run, the job dir is `<YYYY-MM-DD>_stim_<mode>_petsc_np1/`. Verification artefacts:

- `parameters.par` — grep `bidomain`, `num_stim`, `stim\[.\]\.crct`, `stim\[.\]\.pulse` to confirm intended config landed.
- `electrics.log` — solver progress; should run to t=5 ms without errors.
- `vm.igb` — transmembrane voltage time series. Header parseable via the snippet in the **Inspecting IGB** section.
- `phie.igb` — extracellular potential. Present even in `extra_I_mono` (synthesised from monodomain Vm).
- `init_acts_activation-thresh.dat` — only present when LATs are configured (this tutorial does not configure them).

## Inspecting IGB output

`carputils.tools` ships an IGB reader. Quick sanity check (print min/max Vm and final-frame stats) without launching a GUI:

```python
import gzip, os, struct, numpy as np
from carputils.carpio import igb

p = '<job_id>/vm.igb'                          # works for .igb or .igb.gz
with igb.IGBFile(p) as f:
    hdr = f.header()
    data = f.data()                             # shape (T, N)
print(hdr['t'], hdr['x'], data.min(), data.max(), data[-1].mean())
```

For a quick CLI check without writing Python: `igbhead <file>` prints the header (carputils ships `igbutils` under tutorial 05).

## Findings

Populated below as each mode is run.

<!-- FINDINGS_START -->

All six modes were executed at default settings (np=1, `--grounded on`, `tend=5 ms`). Each run completes in ~1 s. Vm and phie were spatially binned into three node groups: **A** (x<−4500 µm, ~20 nodes), **B** (x>4500 µm, ~20 nodes), **C** (|x|<100 µm, ~4 nodes).

Resting Vm = −86.93 mV (DrouhardRoberge with `APDshorten*8`).

### Mode-by-mode results

| mode | phie at A peak | phie at B peak | phie at C peak | Vm at A min | Vm at B max | Vm at C | capture? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `extra_V` | +1956 mV (clamp) | ≈0 (GND) | +1009 (linear midpoint) | −134 mV | **+58 mV** | flat | **B (cathode)** |
| `extra_V_bal` | +956 | −956 | 0 (pinned by C-GND) | −134 | **+119** | small dip to −132 | **B** |
| `extra_V_OL` | +1953 during pulse, ~+33 after | ≈0 | +1009 → +28 after pulse | −134 | **+58** | flat | **B** |
| `extra_I` | +1945 | ≈0 | +1006 | −133 | **+58** | flat | **B** |
| `extra_I_bal` | **−960** | **+939** | 0 (pinned) | (depolarized!) | (hyperpolarized!) | flat | **A (cathode after sign flip)** |
| `extra_I_mono` | +939 | −960 | 0 (pinned) | −133 | **+57** | flat | **B** |

Cross-mode observations:
- `extra_V` ≈ `extra_I` in this geometry: 3.1e6 µA/cm³ was tuned to produce the same ~2 kV drop as the +2 kV voltage clamp. Vm responses match within 1 mV at the peak.
- `extra_V_OL` is identical to `extra_V` during the pulse window (2–4 ms); after the pulse, A floats to ~+33 mV in OL vs. clamping to 0 in V. The Vm difference at A is tiny in this 5 ms window because the extracellular field has already dissipated.
- `extra_V_bal` produces an antisymmetric phie field about x=0 (where C-GND pins φₑ=0). Both ends elicit a strong response: B captures (cathode), A hyperpolarizes deeply.
- `extra_I_bal` mirrors `extra_I` because the intentional negative sign on `stim[0].pulse.strength` makes A the cathode. C-GND pins midline. `crct.balance=0` on B is "auto-balance with stim[0]", which `parameters.par` confirms.
- `extra_I_mono` runs as a balanced bidomain (positive A injection + auto-balanced B + C-GND), mirror of `extra_I_bal`. **See bug below — the monodomain switch never reaches the solver.**

### `--grounded off` not exercised

Skipped per plan. To probe pure-Neumann handling later, run:
```bash
./run.py --stimulus extra_I_bal --grounded off
```
Expected: openCARP detects pure-Neumann config, applies mean-zero phie constraint internally; the C-GND wiring drops out of `parameters.par`.

### Bugs / issues observed in this tutorial

1. **`extra_I_mono` does not actually switch to monodomain.** In `run.py`, the `mod` list correctly accumulates `['-extracell_monodomain_stim', '1', '-bidomain', '0']` for that branch (line 641-642), but the assembly step at line 658 reads `cmd += lstm + rstm + astm` and never appends `mod`. Verified by `--dry`: no `-bidomain 0` ever leaves Python, and the produced `parameters.par` shows `bidomain = 1` with both `phie.igb` and `phie_i.igb` written. To fix, append `+ mod` on line 658.
2. **Stim `crct.balance=0` semantic is non-obvious.** Comment in `run.py:483` says "no strength prescribed, balance with stimulus[0]"; the value `0` is a *flag* selecting auto-balance, not a current value. `carphelp stim+balance` clarifies.
3. **`stimtest.par` initial values for stim[0] (`crct.type=0`, `pulse.strength=0`) are placeholders** that are *always* overridden by the Python lstm/rstm. They show up as the first definition in `parameters.par`, then again with the Python override; openCARP's last-wins semantics mean only the override matters. Useful pattern to remember when reading other tutorials.

### General lessons (transferable to other tutorials)

- carputils' job dirs always contain `parameters.par` reflecting the **effective** (last-wins) configuration. Always grep this file to confirm what actually ran, especially when `.par` and Python both touch the same key.
- `phie.igb` is produced even in (intended) monodomain mode if the simulator was actually told to run bidomain. The presence of `phie_i.igb` (intracellular potential dump) is a clearer bidomain marker than file existence alone.
- Spatially binning by node x-coordinate is a quick verification trick: load `meshes/<auto>/block.pts`, mask by spatial range, average over masked nodes. The strand has 404 nodes in this tutorial.
- The IGB reader returns a flat array; reshape to `(hdr['t'], hdr['x'])`. The `with` form of `IGBFile` is supported but optional — calling `.header()` then `.data()` works on a plain instance.

<!-- FINDINGS_END -->
