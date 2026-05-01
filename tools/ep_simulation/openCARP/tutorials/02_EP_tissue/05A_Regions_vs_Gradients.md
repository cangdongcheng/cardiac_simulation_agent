# CLAUDE.md — 02_EP_tissue / 05A_Regions_vs_Gradients

## What this tutorial demonstrates

Two methods for spatially heterogeneous ionic-model properties on a 2D sheet: **region-wise** (four discrete tag-based regions, each with a different scalar multiplier for GKr and factorGKs) vs. **gradient-wise** (one region, continuous linear gradient via the `-adjustment[]` interface). The tutorial runs both, computes APD from depolarization/repolarization LATs, and writes `DeltaAPD = APD_gradient − APD_region` to show the difference.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 1 cm × 1 cm 2D triangle slab, auto-generated, 100 µm resolution | `run.py:setupMesh()` |
| Ionic model | **MahajanShiferaw** (UCLA rabbit ventricular) | par files |
| Modulated channels | GKr and factorGKs (IKr and IKs) | `run.py:250-256` |
| Scaling range | `--min_scf=0.1` (bottom) to `--max_scf=2.0` (top) | defaults |
| Region-wise steps | 0.1x, 0.733x, 1.367x, 2.0x (4 equally spaced quartiles) | `run.py:251-256` |
| Gradient-wise | linear from 0.1×base to 2.0×base per node via `.adj` file | `run.py:292-322` |
| Stim | transmembrane, left edge (x∈[0,500 µm], full y span) | `run.py:271-278` |
| tend | 300 ms | par file |
| Output | `depol-thresh.dat`, `repol-thresh.dat` → `APD.dat` + `DeltaAPD.dat` | `run.py:343-357` |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/05A_Regions_vs_Gradients
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py
# custom range:
  python run.py --min_scf 0.5 --max_scf 3.0
```

## Findings (run on 2026-04-28)

Two sims, each ~4 min (ionics dominated: 78 s compute at 300 ms tend). 10,201 nodes.

| Y quartile | GKr/GKs scale | APD_region (ms) | APD_gradient (ms) | ΔAPD (ms) |
| --- | --- | --- | --- | --- |
| y<2500 (bottom) | 0.1x | 191.9±4.5 | 188.7±4.4 | −3.2±0.2 |
| 2500–5000 | 0.733x | 183.4±5.5 | 181.9±5.0 | −1.5±0.7 |
| 5000–7500 | 1.367x | 171.0±5.3 | 172.2±4.8 | +1.2±0.7 |
| y>7500 (top) | 2.0x | 162.2±4.2 | 165.2±4.1 | +3.0±0.2 |

Global: APD_region 154.5–200.6 ms; APD_gradient 157.6–197.1 ms; |ΔAPD| max 3.5 ms.

Key observations:
- **APD decreases bottom-to-top** (higher GKr/GKs → faster repolarization → shorter APD). Range ~40 ms across the sheet.
- **ΔAPD is subtle (<4 ms)** even though scaling spans 0.1x–2.0x. The two methods produce very similar spatial APD patterns.
- **ΔAPD sign flips at mid-sheet**: gradient produces shorter APD at the bottom (where it averages more nodes at lower-than-region-step values) and longer at the top. This reflects the region-wise step approximation overshooting the gradient at the bottom and top extremes.
- The boundary-node assignment effect is visible as elevated within-region std at region interfaces.

## Lessons (transferable)

1. **`-adjustment[N].file`** is the openCARP interface for nodal parameter overrides. File format: header `<npts>\nintra\n` then `<node_idx> <value>\n` per line. The companion `.dat` file (one value per line) can be loaded into meshalyzer as a data field.
2. **`adjustment[N].variable`** must match the ionic-model state variable name exposed by `bench --imp-info <model>`. For MahajanShiferaw: `GKr`, `factorGKs`, etc. Use the form `<Model>.<variable>` for model-namespaced variables.
3. **Region-wise staircase vs. gradient**: at the extremes (very low / very high scale), the step approximation is close to the gradient only when the step is narrow. With only 4 steps, the bottom step (0.1x applied to all y<2500) slightly over-represents low-GKr nodes compared to the gradient.
4. **MahajanShiferaw** (alias UCLA_RAB) uses `factorGKs` (not `GKs`) as its IKs scaling parameter. This is model-specific — confirm parameter names via `bench --imp-info`.
5. **APD computation here**: APD = repol_LAT − depol_LAT, where both LATs use threshold = −75 mV. Depol: upstroke crossing (mode=0). Repol: downstroke crossing (mode=1). This is approximately APD_-75 (time from upstroke to 90%+ repolarization for a typical cardiac AP starting at −80 mV rest).
