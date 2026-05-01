# CLAUDE.md — 02_EP_tissue / 08_lats

## What this tutorial demonstrates

Computing **Local Activation Times (LATs)**, **Repolarization Times (REPs)**, and **Action Potential Durations (APDs)** from bidomain tissue simulations. Two configurable threshold crossings are tracked:

- **ACT** (activation): first upward crossing of `-lats[N].threshold` during depolarization
- **REP** (repolarization): first downward crossing of `-lats[N].threshold` during repolarization

APD = REP − ACT at each node.

Supports tracking `vm` (transmembrane voltage) or `phie` (extracellular potential).

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 10×10 mm 2D sheet, `meshes/` | shipped |
| Ionic model | DrouhardRoberge (bidomain) | par file |
| Bidomain | yes (`bidomain=1`) | par file |
| Quantity | `--quantity {vm,phie}` (default `vm`) | CLI |
| ACT threshold | -10 mV (default) | `--act-threshold` |
| REP threshold | -70 mV (default) | `--rep-threshold` |
| tend | 300 ms | par file |
| Stim | 2-stimulus planar S1-S2 from left edge | par file |
| Output | `ACTs.dat`, `REPs.dat`, `APDs.dat` | run.py |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/08_lats
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py

# Different thresholds
python run.py --act-threshold -20 --rep-threshold -60

# Track phie instead of vm
python run.py --quantity phie
```

## Findings (run on 2026-04-28)

Run: `2026-04-28_lats_vm_act-thr_-10mV_rep-thr_-70mV`. Default settings. ~15 s runtime.

### Results

| Quantity | Min (ms) | Max (ms) | Mean (ms) |
| --- | --- | --- | --- |
| ACT | 1.06 | 21.94 | 12.69 |
| REP | 239.21 | 253.50 | 246.64 |
| APD | 231.29 | 238.18 | 233.94 |

**Spatial interpretation**:
- ACT range 1–22 ms reflects wave travel across 10 mm (CV ≈ 10/(22−1)×1000 ≈ 476 mm/s ≈ 48 cm/s — consistent with DrouhardRoberge at default conductivities)
- APD variation ~7 ms (2.9% of APD) across the sheet — small electrotonic effect
- REP tracks closely with ACT pattern (early-activated = earlier repolarization)

## Lessons (transferable)

1. **`-lats[N].measurand`**: `0` = vm, `1` = phie. Set via `--quantity` in this tutorial.
2. **`-lats[N].method 1`** (default): upward threshold crossing for ACT. Method 0 = maximum dVm/dt instead.
3. **`-lats[N].all 0`**: record only the first crossing, not all crossings. Suitable for single-stimulus protocols.
4. **Output format**: `.dat` files have a 1-line header (node count) then one float per line. Load with `np.loadtxt(fname, skiprows=1)`.
5. **APD = REP − ACT** is computed per-node by the post-processing script, not by openCARP directly. Both `.dat` files must exist before computing APDs.
6. **Bidomain LAT tracking on phie**: useful to detect activation from the extracellular potential rather than intracellular — better matches clinical electrogram-based LAT annotations.
7. **Output files**: `phie.igb` and `phie_i.igb` are produced from the bidomain run even when tracking only `vm`. The bidomain field solve is always active when `bidomain=1`.
