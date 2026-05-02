# CLAUDE.md — rules for this library

This file is the constitution for the `cardiac_simulation_agent` library. Both human contributors and agents working in this repo must follow these rules. Read this once before adding or modifying anything.

## Mission

A curated, validated knowledge library that lets an agent — given any question about cardiac simulation in this user's pipeline — find the relevant context in 1-3 hops, apply it correctly, and know when its information is stale.

## Library shape

```
cardiac_simulation_agent/
├── CLAUDE.md            ← this file (rules)
├── README.md            ← human entry: what this is, who it's for
├── INDEX.md             ← agent navigation map (read first for retrieval)
│
├── workflows/           ← GOAL-oriented: tool-chains with stated endpoints
│   └── em-coupling.md
│
├── tools/               ← TOOL-oriented: per-tool usage references, by category
│   ├── ep_simulation/         (openCARP, bench, carputils)
│   ├── mesh_processing/       (meshtool, cobiveco, ...)
│   ├── visualization/         (meshalyzer, ...)
│   └── em_coupling/           (placeholder — FEniCS-pulse, CARPentry-Pro)
│
└── refs/                ← shared bibliography

The library is decoupled from the openCARP install. Tutorial-derived notes are
copies under `tools/ep_simulation/openCARP/tutorials/`. The install at
`/usr/local/lib/opencarp/share/tutorials/` is the place to run tutorials.
```

## Two kinds of notes

| Kind | Lives in | Answers | Endpoints |
|---|---|---|---|
| **Workflow** | `workflows/` | "How do I go from X to Y?" | Inputs and outputs explicitly stated at the top of the note |
| **Tool** | `tools/<category>/` | "How does this tool work?" | Single tool, all aspects (usage, modes, bugs, gotchas) inline |

Concepts, findings, and bugs are **not** separate categories — they live inside whichever workflow or tool note they belong to, as sections.

## What goes IN

- User-specific environment knowledge (paths, conda envs, system quirks)
- Validated experimental results with reproducer (numbers we measured here)
- Synthesis across tools (when to use X vs Y; comparison tables)
- Hard-won lessons (bugs, workarounds, gotchas)
- End-to-end recipes that have been run, with expected outputs and sanity checks
- Distillations of paper material — only when the user keeps re-deriving it AND an agent would otherwise hallucinate; always with citation

## What stays OUT

- Reference material that exists elsewhere and is well-maintained — link, don't rewrite (e.g. openCARP's `--help` output)
- Speculation / aspirational content — if not tested, don't claim
- Verbatim re-snapshots of install-tree `CLAUDE.md` files — `tools/ep_simulation/openCARP/tutorials/` already holds a one-time snapshot; future contributions should be cross-tutorial syntheses or fresh runs, not re-copies
- One-off conversation notes — useful in one chat ≠ reusable
- Code/scripts — the library is documentation; runnable code lives in the openCARP install (`/usr/local/lib/opencarp/share/tutorials/`) or future tool-specific workspaces
- Tutorials for humans — this is a reference, not a textbook

## Validation status (frontmatter)

Every note has a `status:` field:

| Status | Meaning | How to cite |
|---|---|---|
| `validated` | Ran here, reproducer included | "I ran this; here's the result" |
| `mixed` | Some local validation + some borrowed | Distinguish per-claim in the note |
| `borrowed-from-paper` | Derived from publication; not re-tested | Cite the paper, not this library |
| `unverified` | Written but not confirmed | Flag uncertainty to user |
| `TODO` | Placeholder | Should not exist; if you create one, fill it within the same session |

## Frontmatter (required on every note)

```yaml
---
title: Short descriptive title
status: validated | mixed | borrowed-from-paper | unverified
use_when: one-line retrieval hint — when an agent should open this
last_updated: YYYY-MM-DD
---
```

## When a tool gets a folder vs a file

A tool gets its own folder under `tools/<category>/` when:
- It has multiple distinct usage modes (e.g. openCARP: monodomain, bidomain, eikonal, reaction-eikonal)
- It has substantial extensions/bugs that warrant their own files (e.g. meshalyzer + the Haswell shader fix)

Otherwise it's a single `<toolname>.md` at the category level. Default to a flat file; promote to a folder when the file exceeds ~500 lines or has clear sub-modes.

## Workflow note structure

A workflow note must state its endpoints at the top:

```markdown
---
title: ...
status: ...
use_when: ...
last_updated: ...
---

# Workflow name

**Input:**  what you start with (mesh, ionic model, parameters, ...)
**Output:** what you end up with (AP traces, tension, deformation, ...)
**Tools:**  list of tools used, in order

## TL;DR
One paragraph.

## Theory / why
Math and concepts only as needed to understand the chain.

## Steps
Tool-by-tool commands, with expected outputs after each.

## Sanity checks
Numerical values that confirm the run was correct.

## Variations / knobs
Common deviations from the default chain.

## Bugs / gotchas
What broke when we ran this.

## References
Cross-links to tool notes, papers, related workflows.
```

## Tool note structure

```markdown
---
title: ...
status: ...
use_when: ...
last_updated: ...
---

# Tool name

**Install location / how to invoke**
**Where it fits in the pipeline** (e.g. "consumes a CARP mesh, produces an .igb time series")

## TL;DR
One paragraph.

## Common usage
The flags we actually use, with examples.

## Modes / variants
If the tool has multiple modes, document each (or split into sub-files when long).

## Output formats
File layouts, how to read them.

## Bugs / gotchas
Known issues and fixes.

## When to use vs alternatives
Decision criteria — when this tool is right, when something else is better.

## References
Manual links, paper refs, related tools.
```

## File naming

- Lowercase, hyphenated: `em-coupling.md`, `vertex-picking-fixes.md`
- One topic per file
- Tool subfolders: `tools/ep_simulation/openCARP/monodomain.md`
- Avoid abbreviations unless they're the canonical name (`openCARP` is fine; `ec` is not)

## Equations

Unicode + monospace code blocks (terminal-friendly):

```
β · Cm · ∂Vm/∂t = ∇·(D ∇Vm) − β · I_ion(Vm, s) + I_stim
```

LaTeX `$$...$$` allowed in addition when a renderer is expected, but the Unicode block must be present.

## Cross-linking

- Use explicit relative paths: `[tools/visualization/meshalyzer/vertex-picking-fixes.md](../tools/visualization/meshalyzer/vertex-picking-fixes.md)`
- Don't say "see above" — name the file
- Internal anchors (`[section](#section-name)`) are OK within long notes

## Anti-rot rules

- Update `last_updated` whenever you change a note's content
- When you change a tool/install, update or mark stale all notes that reference it
- Don't create empty stubs. If a directory has nothing yet, its README mentions it as a "likely addition" but no file is created
- If a note has been wrong twice, retire it — better to have less and trust it than more and hedge

## Agent responsibilities

When working in this library:

1. **Read `INDEX.md` first** for retrieval. It's optimized for that.
2. **Check frontmatter `status`** before citing. `validated` claims can be quoted directly; `borrowed-from-paper` should cite the paper as authority.
3. **When you learn something new, update or add a note** — don't leave knowledge only in the conversation. Match the new content against the workflow vs tool taxonomy, place it accordingly.
4. **When you change anything, update `last_updated`** in the affected note's frontmatter.
5. **Don't duplicate** — if the same fact is needed in two places, put it in one note and cross-link from the other.
6. **No empty stubs** — if you can't fill it now, don't create the file. The directory's README can mention "future" content.

## Scope boundary

| | In scope | Case-by-case | Out (for now) |
|---|---|---|---|
| Single-cell EP | ✓ | | |
| Tissue EP (mono/bidomain/eikonal) | ✓ | | |
| Mechanics (passive + active) | ✓ | | |
| EM coupling | ✓ | | |
| Standard ionic models | ✓ | | |
| Visualization | ✓ | | |
| Mesh processing & fiber generation | ✓ | | |
| Hemodynamics (0D windkessel) | | ✓ | |
| FSI (blood flow) | | ✓ | |
| Image-based personalization | | ✓ | |
| Clinical workflows | | ✓ | |
| ML surrogates | | ✓ | |
| Generic FEM theory | | | ✗ — refer to textbooks |
| Software engineering | | | ✗ |
| Non-cardiac simulation | | | ✗ |

The case-by-case row: include only when a concrete experiment or question forces us to.
