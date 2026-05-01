# cardiac_simulation_agent

A validated knowledge library for cardiac simulation, structured for an agent to pick up the entire pipeline quickly. Distills hands-on experience with openCARP, single-cell modeling, and electromechanical coupling theory into atomic, retrievable notes.

## Who this is for

- An LLM agent that needs to answer questions or run experiments in cardiac EP / mechanics / coupling
- A human collaborator who wants a single place to find "what tool to use", "how to run X", "what does the math look like", and "what bug should I expect"

## Layout

```
cardiac_simulation_agent/
├── README.md                    ← you are here (human entry)
├── INDEX.md                     ← agent navigation map
│
├── concepts/        ← WHAT     theory, math, physics
├── tools/           ← WITH WHAT software references (openCARP, bench, ...)
├── workflows/       ← HOW      end-to-end recipes
├── findings/        ← LEARNED  validated results from experiments
├── troubleshooting/ ← FIX      bug catalog and known-good patches
├── refs/            ← EXTERNAL papers, manuals, links
│
└── opencarp_tutorials → /usr/local/lib/opencarp/share/tutorials
                                ← write-through symlink to the install
```

## Conventions

### Frontmatter (every note)

```yaml
---
title: ...
status: validated | mixed | borrowed-from-paper | unverified | TODO
use_when: one-line retrieval hint — when an agent should open this
last_updated: YYYY-MM-DD
---
```

`status` semantics:
- **validated** — claims tested or commands run on this machine
- **mixed** — some claims validated locally, some borrowed
- **borrowed-from-paper** — derived from publications/textbooks; not re-tested
- **unverified** — written but not confirmed; flag for follow-up
- **TODO** — placeholder

### Per-directory READMEs

Every subdirectory has a `README.md` listing its contents with one-line descriptions. An agent entering a directory reads that file first.

### File naming

- Lowercase, hyphenated: `em-coupling.md`, not `EM_Coupling.md`
- One topic per file; split rather than grow monoliths beyond ~500 lines
- Tutorial-derived findings live in `findings/`, not duplicated into individual tutorial dirs

### Equations

Unicode + monospace code blocks (terminal-friendly). LaTeX `$$...$$` allowed in addition if a renderer is expected.

## Adding new notes

1. Pick the right directory based on the WHAT/HOW/WITH WHAT/LEARNED/FIX classification.
2. Write a self-contained note. Start with frontmatter, then a one-paragraph TL;DR.
3. Append a one-line entry to the directory's `README.md`.
4. If the note belongs in retrieval shortcuts, add a row to top-level `INDEX.md`.

## Local environment

- openCARP install: `/usr/local/bin/openCARP`, `bench`, `meshtool`
- Meshalyzer: `/usr/local/meshalyzer` (patched, see `troubleshooting/meshalyzer-haswell.md`)
- carputils Python: `/home/cdc/miniconda3/envs/opencarp/bin/python`
- Tutorial source tree: `opencarp_tutorials/` (symlinked) — write-through

The `opencarp` conda env must be on `PATH` for tools like `tuneCV`, `restituteCV`, `cellml_converter.py` to resolve correctly.
