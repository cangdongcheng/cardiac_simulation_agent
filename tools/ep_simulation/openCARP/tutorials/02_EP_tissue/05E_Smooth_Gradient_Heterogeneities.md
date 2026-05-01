# CLAUDE.md — 02_EP_tissue / 05E_Smooth_Gradient_Heterogeneities

## What this tutorial demonstrates

**Nodal-level** assignment of ionic-model parameters via the `adjustment[]` interface, creating smooth (continuous, linear) 2D gradients across the sheet. Two independent gradients are applied simultaneously: one along X (left-to-right) and one along Y (bottom-to-top). Default: GKr 0x→5x left-to-right, Gto 0x→5x bottom-to-top.

Uses `tenTusscherPanfilov` (human ventricular) — different from 05A (MahajanShiferaw). The adjustable parameters must be "state variables" exposed by `bench --imp-info tenTusscherPanfilov`: GCaL, GKr, GKs, Gto.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | Auto-generated 1 cm × 1 cm tri slab, 100 µm, 4 tags | `run.py:setupMesh()` |
| Ionic model | tenTusscherPanfilov | par file |
| Default X-gradient | GKr, 0x→5x left→right | `--xgrad_var GKr --xgrad_left_scf 0.0 --xgrad_right_scf 5.0` |
| Default Y-gradient | Gto, 0x→5x bottom→top | `--ygrad_var Gto --ygrad_bottom_scf 0.0 --ygrad_top_scf 5.0` |
| tend | 500 ms | par file |
| Output | `vm.igb`, `init_acts_depol-thresh.dat`, `init_acts_repol-thresh.dat`, `APD.dat` | par + run.py |
| Conductivity | g_il=1.0, g_it=0.2, g_in=0.2 S/m (uniform) | par file |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/05E_Smooth_Gradient_Heterogeneities
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py

# Flip gradient direction
python run.py --xgrad_flip --ygrad_flip

# Different variables (don't set xgrad_var == ygrad_var)
python run.py --xgrad_var GKs --ygrad_var GCaL
```

**Note**: `--xgrad_var` and `--ygrad_var` must not be the same variable.

## Adjustment file format (`.adj`)

```
<npts>
intra
<node_idx> <value>
<node_idx> <value>
...
```

Variable name format for ionic model state variables: `<ModelName>.<variable>`, e.g., `tenTusscherPanfilov.GKr`. The script writes this in the command via `-adjustment[N].variable tenTusscherPanfilov.GKr`.

## Findings (run on 2026-04-28)

500 ms simulation, ~4 min run time. Mesh: 10,201 nodes. APD range: 227.3–272.4 ms.

### APD by X-column (GKr gradient 0x→5x, left→right)

| X range (µm) | GKr scale | Mean APD (ms) | Std (ms) |
| --- | --- | --- | --- |
| 0–2500 (left) | ~0x | 269.0 | 2.3 |
| 2500–5000 | ~1.7x | 258.7 | 4.2 |
| 5000–7500 | ~3.3x | 243.5 | 4.3 |
| 7500–10000 (right) | ~5x | 232.1 | 2.5 |

**GKr↑ → shorter APD**: IKr is a major repolarizing current in tenTusscherPanfilov. 0x→5x GKr range produces ~37 ms APD spread across the 1 cm sheet.

### APD by Y-row (Gto gradient 0x→5x, bottom→top)

| Y range (µm) | Gto scale | Mean APD (ms) | Std (ms) |
| --- | --- | --- | --- |
| 0–2500 (bottom) | ~0x | 250.0 | 13.2 |
| 2500–5000 | ~1.7x | 251.1 | 14.2 |
| 5000–7500 | ~3.3x | 251.1 | 15.1 |
| 7500–10000 (top) | ~5x | 250.4 | 15.7 |

**Gto↑ has minimal effect on APD** in tenTusscherPanfilov: Ito (transient outward) mainly shapes the early repolarization notch (phase 1) but does not substantially shorten the plateau. APD mean is essentially constant across rows (250.0–251.1 ms). The increasing std with y reflects that cells with very high Gto show a deeper notch → slightly variable depol/repol LAT detection.

### Key take-away

The X and Y gradients are orthogonal effects: the GKr gradient produces a clear left-to-right APD gradient; the Gto gradient mostly modifies AP shape (notch depth) without substantially changing APD in this model. Visualized in vm.igb, repolarization proceeds right-to-left (high-GKr side repolarizes first).

## Lessons (transferable)

1. **The `adjustment[]` interface** is the nodal-level equivalent of `imp_region[].im_param`. It overrides a single ionic model state variable at each node independently.
2. **Only "state variables" that are also "parameters"** (i.e., not time-evolving ODEs) can be adjusted nodally. Confirm with `bench --imp-info <model>`. For tenTusscherPanfilov: GCaL, GKr, GKs, Gto are eligible.
3. **GKr is a stronger APD modulator than Gto** in human ventricular models: GKr is a plateau current (active during phase 2–3); Gto is an early repolarization current (active during phase 1 only in epicardial cells).
4. **Base values** for tenTusscherPanfilov: GCaL=3.98e-5, GKr=0.153, GKs=0.392, Gto=0.294 (from `get_gbase()` in run.py:305-310). The adjustment values multiply these.
5. **tend=500 ms** ensures full AP cycle including repolarization is captured at all nodes, even in high-APD regions.
