# CLAUDE.md — 02_EP_tissue / 03B_study_prep_init

## What this tutorial demonstrates

How to **initialize a tissue model** with pre-computed single-cell state variables to avoid running hundreds of beats in the full tissue. Three methods are compared:

1. **`none`** — openCARP hardcoded defaults for the chosen ionic model.
2. **`sv_init`** — state variables from a `bench` single-cell pacing run (file `INIT.sv`) applied uniformly to all cells.
3. **`prepace`** — like `sv_init` but each cell's state is taken at a time offset based on a pre-known activation sequence (`LATs.dat`), so different cells start at different points in the cycle.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 10×10 mm 2D sheet, dx=0.2 mm, `hexa` elements | `run.py:241` |
| Ionic model | `DrouhardRoberge` (default) or `tenTusscherPanfilov` | `--model` |
| Solver | monodomain (default) | `tools.carp_cmd()` |
| Conductivities | isotropic: g_il=g_it=0.174, g_el=g_et=0.625 S/m | `run.py:281-284` |
| Stim | transmembrane, 100 µA/cm², 1 ms at t=0, left edge x∈[0,0.5 mm] full y | `run.py:287-295` |
| tend | 50 ms | `run.py:299` |
| dt | 100 µs | `run.py:298` |
| spacedt | 1 ms | `run.py:301` |
| `--bcl` | 500 ms (for bench pacing in sv_init) | parser default |
| `--npls` | 10 pulses | parser default |

For `sv_init`: bench runs `npls × bcl` = 5000 ms simulation → saves `INIT.sv`. Fast (<1 s) for single-cell.
For `prepace`: uses `LATs.dat` (pre-existing file) as activation sequence; openCARP shifts each cell's state by its LAT offset.

## CLI

```bash
# Default init (hardcoded model defaults)
./run.py

# sv_init: pre-pace with bench first, then init all cells identically
./run.py --init-method sv_init

# prepace: spatially-varying init based on LATs.dat
./run.py --init-method prepace

# Change model
./run.py --init-method sv_init --model tenTusscherPanfilov
```

## Relevant openCARP parameters

```
-imp_region[0].im_sv_init  INIT.sv        # for sv_init
-prepacing_beats            10             # for prepace
-prepacing_lats             LATs.dat       # for prepace
-prepacing_bcl              500            # for prepace
```

## Findings (run on 2026-04-28)

All three runs completed in <1 s electrics compute.

| run | t=0 Vm (mean / range) | note |
| --- | --- | --- |
| `none` | −86.9269 / [−86.9269, −86.9269] mV | hardcoded DrouhardRoberge default; all cells identical |
| `sv_init` | −85.7103 / [−85.7103, −85.7103] mV | post-pacing rest; uniform; ~1.2 mV higher than default |
| `prepace` | −85.6684 / [−85.7108, −85.6336] mV | spatially varying ±0.077 mV spread; left edge starts slightly ahead |

End-of-sim Vm at t=50 ms (wave propagating):
- `none`: mean 9.68 mV range [9.26, 9.98] — slightly behind sv_init/prepace
- `sv_init`: mean 11.54 mV range [10.54, 11.91] — slightly ahead
- `prepace`: mean 11.32 mV range [10.18, 11.82] — similar to sv_init

The Vm differences at t=0 are small (~1 mV), but other state variables (e.g. intracellular Ca_i = 0.30 for `none` vs 0.172 for `sv_init` at BCL=500) differ substantially. Over time this affects APD, CV, and restitution behavior — hence the importance of proper initialization for quantitative studies.

The key insight: `none` starts with the ionic model's *arbitrary* default initial conditions, which may be far from the steady-state limit cycle. `sv_init` is computationally cheap (bench runs single-cell) and gives a much better starting point. `prepace` is the most biologically realistic — it accounts for the fact that different cells in the tissue are at different phases of their cycle depending on the activation sequence.

## Lessons (transferable)

1. **`-imp_region[0].im_sv_init`** takes a path to a `.sv` file produced by `bench --save-ini-file`. The file contains one initial condition value per ODE state variable per cell (or a single set applied uniformly).
2. **`-prepacing_beats / -prepacing_lats / -prepacing_bcl`** are the three prepacing parameters. `LATs.dat` is a text file with one local activation time per node (in ms). openCARP offsets each cell's state-variable read by its LAT.
3. **Bench `--save-ini-time tend`** saves the state at the *end* of the simulation — i.e., at the end of the last beat. If you want to save at a specific phase, adjust `--save-ini-time`.
4. **2D sheet generation**: `mesh.Block` + `corner_at_origin()` + `BoxRegion` + `mesh.generate()` is the standard 2D recipe. The mesh origin ends up at the lower-left corner after `corner_at_origin()`, so coords are in [0, wedge_x] × [0, wedge_y].
5. **Stim at left edge**: electrode box uses x∈[0, 500 µm], full y span. The Python code uses `(wedge_y/2*1000)-250` and `5250` for y, which is the lower half + overlap in a 10 mm sheet (y range in µm = [4750, 5250] ≈ center row). Actually this is the stim site at x=0, y≈5 mm midline.
