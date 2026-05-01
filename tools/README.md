# tools/

**Tool-oriented** notes: per-tool usage references, organized by category.

Each tool note answers: how to invoke, key flags, output formats, common modes, known bugs, when to use vs alternatives. See `../CLAUDE.md` § "Tool note structure" for the required layout.

## Categories

| Subdir | Purpose |
| --- | --- |
| `ep_simulation/` | Solvers for cellular and tissue electrophysiology — openCARP, bench, carputils |
| `mesh_processing/` | Geometry preparation, meshing, fiber assignment — meshtool, cobiveco |
| `visualization/` | Trace and field visualizers — meshalyzer, limpetGUI, paraview |
| `em_coupling/` | Tools that solve mechanics or couple EP↔mechanics — placeholder; populated when FEniCS-pulse / CARPentry-Pro / Abaqus enter the pipeline |

## Folder vs file

A tool gets its own folder when it has multiple distinct usage modes (openCARP) or substantial extensions/bugs that warrant their own files (meshalyzer + Haswell fix). Otherwise a single `<toolname>.md` at the category level. See `../CLAUDE.md` § "When a tool gets a folder vs a file".
