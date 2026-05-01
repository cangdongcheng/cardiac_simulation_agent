# tools/

The **WITH WHAT**: per-tool reference notes. How to install, invoke, and reason about each piece of software in the cardiac simulation pipeline.

A note belongs here if it answers "what flags does this tool accept", "what does its output look like", "how does it relate to other tools".

## Current notes

(none yet — add as questions arise)

## Likely additions

- `opencarp.md` — invocation patterns, .par files, monodomain vs bidomain, parameter dictionary lookup with `carphelp`
- `bench.md` — single-cell solver, output formats, `--bin` vs ASCII, all the experiment types (pacing, restitution, voltage clamp, EM)
- `carputils.md` — Python wrapper, `@tools.carpexample`, parser conventions, common pitfalls
- `meshtool.md` — mesh manipulation, format conversion, fiber assignment
- `meshalyzer.md` — visualization, .mshz views, vertex picking, links to `troubleshooting/meshalyzer-haswell.md`
- `limpetGUI.md` — single-cell trace visualization
- `ecosystem.md` — FEniCS-pulse, Chaste, Cardioid, CARPentry-Pro (NumeriCor), Abaqus + Living Heart, Continuity. Comparison table.

Each per-tool note should at minimum cover: install location, key flags, output formats, common gotchas, links to relevant `concepts/`, `workflows/`, and `troubleshooting/` notes.
