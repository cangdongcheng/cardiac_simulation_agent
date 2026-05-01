# CLAUDE.md — 02_EP_tissue / 07B_periodic

## What this tutorial demonstrates

**Periodic boundary conditions (PBCs)** implemented as high-conductivity line elements connecting opposite tissue edges. Connects left↔right (X-periodic) and/or bottom↔top (Y-periodic) — effectively turning a 2D sheet into a cylinder or torus. PBCs allow spiral waves to persist indefinitely without annihilating at boundary edges, approximating closed cardiac surfaces.

The implementation uses `mesh.Block(periodic=...)` to generate connection elements (tagged 1234=X, 1235=Y) and assigns conductivity `g_il = cnnxG` (default 100,000 S/m) to short-circuit the opposing edges.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | square sheet, default 4 cm × 4 cm, 2D | `--size` |
| Ionic model | DrouhardRoberge, `APDshorten*8` | run.py:243 |
| BCs | `--bc {noflux,Xper,Yper,XYper}` | CLI |
| Connection G | 100,000 S/m default | `--cnnxG` |
| S1 stim | horizontal strip at y≈0, full-width | run.py |
| S2 stim (optional) | quarter-sheet cross-shock | `--S2-start` |
| Block stimulus | Vm clamp via `stimtype=9` to prevent propagation across edge during S2 | `--block-Vm`, `--block-dur` |
| tend | 1000 ms default | `--tend` |

## Status: Fixed — `model.Stimulus` API patch applied

The tutorial was previously broken because `run.py` called `model.Stimulus` with legacy `stimulus[]` geometry kwargs (`x0/y0/xd/yd`) that the current `model.Stimulus` class rejects. The fix replaces all four `model.Stimulus` calls with a local `stim_box` helper that emits raw `-stim[N].*` CLI flags directly, bypassing the class validator entirely.

**Patch summary (run.py:211–258):**
- Introduced `stim_box(idx, name, x0, y0, xd, yd, strength, duration, start, crct_type=0, ...)` helper that builds a flat list of `-stim[N].key value` pairs.
- Geometry specified via `elec.p0[0/1/2]` / `elec.p1[0/1/2]` indexed scalars (not the old `x0/xd/yd`).
- Renamed kwargs: `strength` → `pulse.strength`, `duration` → `ptcl.duration`, `start` → `ptcl.start`, `stimtype` → `crct.type`.
- Block stimuli (Yblock/Xblock, `crct.type=9`) additionally set `pulse.tau_edge=0` and `pulse.tau_plateau=1e6` via optional kwargs.
- All four stimulus definitions (S1, optional S2, optional Xblock/Yblock) now use `stim_box`.

The tutorial runs correctly with the patched file.

## Findings: Run results

### Exp 1 — no-flux, no periodic BCs (baseline)

**Command:** `--np 1 --bc noflux --tend 300 --size 2`  
**Output dir:** `2026-04-28_periodic_300.0_np_1_noflux_2.0/`  
**Wall time:** ~3 min 20 sec (200 s total: 160 s electrics + 40 s ionics)  
**Mesh:** 40,401 nodes, 40,000 quad elements, 2 cm × 2 cm sheet, 100 µm resolution  
**Stimulus:** S1 horizontal strip at y ≈ −9802 to −9402 µm (near bottom edge), full x-width  
**Wave behaviour:** S1 fires at t=0, propagates as a planar wave from the bottom toward the top of the sheet. With no-flux BCs the wavefronts annihilate at the top edge. Single AP propagation; tissue at rest by t ≈ 200–250 ms. No reentry. Output: `vm.igb` (47 MB, 301 time frames × 40,401 nodes, 1 ms/frame).

### Exp 2 — Y-periodic with Vm clamp block (reentry attempt)

**Command:** `--np 1 --bc Yper --tend 500 --size 2 --block-dur 90 --block-Vm -80`  
**Output dir:** `2026-04-28_periodic_500.0_np_1_Yper_2.0/`  
**Wall time:** ~6 min 3 sec (363 s total: 298 s electrics + 65 s ionics)  
**Mesh:** 40,401 nodes, 40,000 quads + 201 Y-periodic line connection elements (tag 1235, g_il = 100,000 S/m). Mesher confirms `Number of periodic connections: 201`.  
**Stimuli:** S1 (same as Exp 1) + Yblock (`crct.type=9`, Vm clamped to −80 mV, 90 ms duration from t=0) holding a thin strip along the bottom edge, blocking wavefront from crossing via the periodic Y connection during the S1 propagation phase.  
**Wave behaviour:** Yblock clamp prevents the S1 wavefront from re-entering through the Y-periodic link for 90 ms. After block lifts at t=90 ms, a unidirectional wave re-enters through the periodic Y connection where tissue has recovered. `Yblock.trc` confirms block active from t=0 to t=90 ms (value = 1.0 throughout). The tutorial's suggested parameters (`--block-Vm -80 --block-dur 90`) produce a scenario consistent with sustained Y-periodic re-entry on a 2 cm sheet. Output: `vm.igb` (78 MB, 501 time frames × 40,401 nodes, 1 ms/frame).

## Conceptual summary

### Periodic connection parameters

| `--bc` | mesh.Block `periodic` | Added regions | `gregion` used |
| --- | --- | --- | --- |
| `noflux` | 0 | none | none |
| `Xper` | 1 | tag 1234 | g_il=cnnxG |
| `Yper` | 2 | tag 1235 | g_il=cnnxG |
| `XYper` | 3 | 1234 + 1235 | g_il=cnnxG (both) |

### Reentry induction with PBCs

Cross-shock protocol fails with PBCs (S2 goes everywhere). Two alternatives:
1. Run initial phase on non-periodic mesh, save state, restart on periodic mesh just after S2 fires.
2. Use a `stimtype=9` (Vm clamp) block line to temporarily block propagation across one edge during S2, then release.

## Lessons (transferable)

1. **Periodic BCs via high-G links**: the solver itself doesn't have native PBC support; the workaround is inserting 1D elements with giant conductivity. For `cnnxG=1e5 S/m`, voltage drop across the link is negligible.
2. **Region tags 1234/1235** are hardcoded in the mesher for X/Y periodic connections respectively. Always use a separate `gregion[]` with these tags to assign the high conductivity.
3. **`stimtype=9`** is the Vm clamp stimulus type — it holds transmembrane voltage at `strength` mV for `duration` ms, creating a transient conduction block line.
4. **`stim_box` helper pattern**: when `model.Stimulus` lacks support for needed kwargs, bypass it entirely by emitting raw `-stim[N].key value` pairs. This avoids the class validator and gives full control over the openCARP `stim[]` parameter namespace.
