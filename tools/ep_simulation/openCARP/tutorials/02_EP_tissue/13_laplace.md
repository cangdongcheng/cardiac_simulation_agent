# CLAUDE.md — 02_EP_tissue / 13_laplace

## What this tutorial demonstrates

Running openCARP in **Laplace-solve-only mode** (`-experiment 2`) to compute a normalized potential field [0, 1] across a tissue domain. No time integration occurs — only the elliptic PDE is solved once with fixed boundary conditions. The result is a steady-state potential distribution driven by voltage BCs at two edges (Dirichlet: 0 V at one end, 1 V at the other). Common use case: computing fiber coordinate systems (e.g., φ for transmural direction) from mesh geometry alone.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | 10×10 mm 2D square slab, 100 µm resolution | run.py (auto-generated) |
| Nodes | 20,402 | mesh |
| `experiment` | 2 (Laplace only) | par flag |
| `bidomain` | 1 (required for elliptic solve) | run.py |
| BCs | `stim[0]`: ground (crct.type=3) on left edge; `stim[1]`: voltage 1 V (crct.type=2, strength=1) on right edge | parameters.par |
| Conductivity | g_il=g_it=g_el=g_et=1 S/m (isotropic, uniform) | parameters.par |
| Output | `phie.igb`, `phie_i.igb` | |

## CLI

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/13_laplace
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py
```

No significant CLI arguments beyond standard carputils flags. The mesh and boundary conditions are hardcoded in `run.py`.

## Findings (run on 2026-04-28)

Output directory: `2026-04-28_laplace/`. Files: `phie.igb`, `phie_i.igb`, `vm.igb` (trivially zero for Laplace-only), `parameters.par`.

`phie.igb` contains the normalized potential field (0 at left edge, 1 at right edge). The field is spatially smooth and monotonically increases left-to-right. With isotropic uniform conductivity and a rectangular mesh, the solution is simply φ(x) = x/L (linear gradient).

Run completed in < 5 s (elliptic solve only, no ODE integration).

## Lessons (transferable)

1. **`-experiment 2`** skips time integration entirely and runs only the Laplace (elliptic) solver. No ionic model is evaluated. This is the standard way to compute steady-state potential fields (fiber coordinates, bath potentials, lead fields).
2. **`bidomain 1` is required** for experiment 2 — the elliptic PDE is the bidomain elliptic component. Even if your target is an intracellular field, you must enable bidomain.
3. **Dirichlet BCs via stimuli**: 
   - `stim[N].crct.type = 3` → ground (φ = 0) applied at specified electrode
   - `stim[N].crct.type = 2` + `stim[N].pulse.strength = V` → fixed voltage V applied
   - The electrode box (p0/p1 coordinates) specifies the surface where the BC is applied
4. **Output interpretation**: `phie.igb` is the extracellular potential (φ_e). In a pure Laplace solve without ionic sources, `phie_i.igb` (intracellular) equals `phie.igb` (no transmembrane voltage gradient). `vm.igb` = φ_i − φ_e ≈ 0.
5. **Use case in fiber coordinate computation**: run Laplace from apex to base → gives transmural/apicobasal coordinate at every node. Commonly used in rule-based fiber generation pipelines (before running the actual EP simulation).
6. **Conductivity anisotropy in Laplace**: if `g_il ≠ g_it`, the isopotential lines follow conductivity-weighted geodesics rather than straight lines. Useful for fiber-following coordinates.
