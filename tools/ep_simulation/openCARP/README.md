# tools/ep_simulation/openCARP/

openCARP is the open-source cardiac EP simulator. Multiple solver modes are exposed via the same binary, controlled by `.par` parameters.

**Install:** `/usr/local/bin/openCARP`
**Source tree:** `/usr/local/lib/opencarp/share/openCARP/`
**Manual:** <https://opencarp.org/manual/opencarp-manual-latest.pdf>
**Tutorial tree:** `../../../opencarp_tutorials/`

## Solver modes (each gets its own note)

| File | Status | Mode |
| --- | --- | --- |
| (none yet) | | |

Likely additions:

- `monodomain.md` — single PDE for Vm; default mode for tissue propagation studies
- `bidomain.md` — coupled φi/φe equations; needed for extracellular potentials, defibrillation
- `eikonal.md` — `--experiment 4` activation-time-only solver
- `reaction-eikonal.md` — eikonal + reaction; faster than monodomain for large meshes
- `pseudo-bidomain.md` — `phie_recovery` post-processing for monodomain → φe estimate

## Cross-cutting topics that may need their own notes

- `parameter-dictionary.md` — how to use `carphelp` to discover flags; `--dry`-and-grep workflow
- `mesh-and-tags.md` — element tag conventions (1=tissue, 1234/1235=periodic links, etc.)
- `output-formats.md` — `.igb`, `.dat`, state-vector files, `parameters.par` as ground truth

When `openCARP/README.md` itself grows beyond a few-line index, split content into the files listed above.
