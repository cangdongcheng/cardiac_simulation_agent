# tools/visualization/

Tools for visualizing cardiac simulation outputs: trace plots, mesh fields, and animations.

## Current notes

| File | Status | Tool |
| --- | --- | --- |
| `meshalyzer/haswell-fix.md` | validated | Patch for meshalyzer shader compilation + vertex picking on Intel Haswell / WSL |

## Likely additions

- `meshalyzer/README.md` — meshalyzer general usage: opening meshes + .igb time-series, .mshz views, vertex picking, animation controls. Install: `/usr/local/meshalyzer/meshalyzer`.
- `limpetGUI.md` — single-cell trace visualization. Used after `sv2h5b` / `sv2h5t` conversion of bench outputs. Install: in `opencarp` conda env.
- `paraview.md` — when meshalyzer isn't enough (e.g. publication-quality figures, custom colormaps, animations beyond `.mshz`).
- `python-plotting.md` — if we develop a recipe for plotting `.igb` / `.bin` outputs in matplotlib.

Tool-specific bug fixes (like `meshalyzer/haswell-fix.md`) live alongside the tool's README in its folder.
