# tools/mesh_processing/

Geometry preparation: meshing, format conversion, fiber assignment, region tagging.

## Current notes

(none yet)

## Likely additions

- `meshtool.md` — openCARP's mesh manipulation tool. Format conversion, element extraction, refinement, mesh statistics. Install: `/usr/local/bin/meshtool`.
- `cobiveco.md` — cobiveco rule-based ventricular fiber and coordinate assignment (Bayer–Verschure–CoBiVeCo method). Used by the user previously; document install + invocation when written.
- `gmsh.md` — generic 3D meshing, when needed for non-rule-based geometries.
- `mesh-fibers-pipeline.md` — could go here as a tools-side reference, or as a workflow under `workflows/` (it's a chain — likely workflow). Decide when written.

When the same conceptual operation is exposed by different tools (e.g. fiber generation via cobiveco vs LDRB vs anatomically-based), the comparison goes here as a "when to use vs alternatives" section in each tool's note.
