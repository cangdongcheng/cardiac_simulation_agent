# INDEX — agent navigation map

Compact sitemap. Read this once to plan retrieval, then open the relevant note(s).

## Two doors

| Question shape | Go to |
| --- | --- |
| "How do I go from X to Y?" / "How do I run a Y experiment?" | `workflows/` |
| "How does tool Z work?" / "Which flag for Z?" / "Z is broken" | `tools/<category>/` |
| "Where's the source paper for X?" | `refs/` |
| Need to actually run a tutorial | `/usr/local/lib/opencarp/share/tutorials/` (the install; or `~/tutorials`) |
| Tutorial-derived findings (curated) | `tools/ep_simulation/openCARP/tutorials/` |

Concepts, findings, and bugs are NOT separate categories — they live as sections inside the relevant workflow or tool note.

## Tool categories

| Subdir | Purpose | Folder if any |
| --- | --- | --- |
| `tools/ep_simulation/` | EP solvers and orchestration | `openCARP/` (multi-mode) |
| `tools/mesh_processing/` | Geometry, meshing, fibers | |
| `tools/visualization/` | Trace and field visualizers | `meshalyzer/` (haswell fix lives here) |
| `tools/em_coupling/` | Mechanics solvers, EP↔mechanics coupling tools | (placeholder) |

## Current notes

| Path | Status | Use when |
| --- | --- | --- |
| `workflows/em-coupling.md` | mixed | Question about Vm → Ca → tension → deformation; choosing an EM tool |
| `tools/visualization/meshalyzer/vertex-picking-fixes.md` | validated | Meshalyzer shader errors or broken vertex picking on Intel Haswell / WSL / Windows Cygwin |

## Environment shortcuts

```
openCARP            /usr/local/bin/openCARP
bench               /usr/local/bin/bench
meshtool            /usr/local/bin/meshtool
meshalyzer          /usr/local/meshalyzer/meshalyzer  (patched)
carputils python    /home/cdc/miniconda3/envs/opencarp/bin/python

PATH-required tools (need opencarp env on PATH):
  tuneCV, restituteCV, cellml_converter.py, limpet_fe.py, make_dynamic_model.sh
```

## Tutorial knowledge

Per-tutorial notes are in the library under `tools/ep_simulation/openCARP/tutorials/<section>/<tutorial>.md`. Top-level summaries:

- `tools/ep_simulation/openCARP/tutorials/01_EP_single_cell/PROGRESS.md` — 9 single-cell tutorials swept
- `tools/ep_simulation/openCARP/tutorials/02_EP_tissue/PROGRESS.md` — 23 tissue tutorials swept
- `tools/ep_simulation/openCARP/README.md` — index linking each tutorial note with its headline finding

The originals also remain in the install at `/usr/local/lib/opencarp/share/tutorials/<section>/<tutorial>/CLAUDE.md` for reference while running tutorials in place.

When digesting cross-tutorial knowledge into the library, place it in the most appropriate workflow or tool note as a section, not a separate file.

## Retrieval tips for agents

- Match the user's question against `use_when:` frontmatter — that's the primary retrieval signal.
- A directory's `README.md` is its index — open before diving into individual notes.
- `status: validated` notes can be cited directly. `status: borrowed-from-paper` should cite the paper as authority, not this library.
- If the answer would require updating a note, do it (per `CLAUDE.md` § "Agent responsibilities").
- If a note is missing, add it under the right directory rather than embedding the answer only in the conversation.
