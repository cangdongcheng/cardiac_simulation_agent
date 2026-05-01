# tools/em_coupling/

**Placeholder.** Tools that solve mechanics or couple EP↔mechanics will live here when they enter the user's pipeline.

Currently this directory is empty by design — the user has openCARP (EP-only) and bench (single-cell EP+stress plug-ins). No standalone mechanics solver is installed.

## Likely future additions

- `fenics-pulse.md` — FEniCS-based hyperelasticity library focused on cardiac mechanics. Free, open-source. Install: `pip install fenics-pulse`.
- `carpentry-pro.md` — commercial successor of CARPentry, sold by NumeriCor GmbH. Multiscale-multiphysics: EP + active stress + passive mechanics + 0D windkessel. Input-compatible with openCARP.
- `chaste.md` — Oxford-developed open-source full-EM platform. Different input format from openCARP.
- `abaqus-living-heart.md` — Dassault SIMULIA Living Heart Project. Commercial, regulatory-grade.

## Background

Until a mechanics solver is in place, EM coupling work happens via:
1. **Single-cell EM with bench** — see `../ep_simulation/` (when bench.md is written) and the `06_EM_coupling` tutorial.
2. **Prescribed deformation** — feed bench/openCARP a `λ(t)` trace from imaging or an external mechanics simulation, and study the EP response. Cheap workaround for many EM questions.

For the EP→mechanics coupling pipeline at the conceptual level, see `../../workflows/em-coupling.md`.
