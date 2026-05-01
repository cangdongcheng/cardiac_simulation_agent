# tools/ep_simulation/

Solvers and orchestration tools for cardiac electrophysiology — single-cell to tissue.

## Current notes

(none yet — populate as we document each tool)

## Likely additions

- `openCARP/` — folder, because openCARP has multiple solver modes:
  - `monodomain.md` — most common tissue EP mode
  - `bidomain.md` — when extracellular potentials matter (defibrillation, electrode studies)
  - `eikonal.md` — fast wavefront-only approximation
  - `reaction-eikonal.md` — eikonal with reaction term retained
- `bench.md` — single-cell solver. Pacing, voltage clamp, restitution, EM via stress plug-ins.
- `carputils.md` — Python wrapper that orchestrates openCARP and bench runs. Parser conventions, `@tools.carpexample`, common pitfalls.

When documenting these, draw from the per-tutorial notes in `openCARP/tutorials/` for validated examples.
