# workflows/

The **HOW**: end-to-end recipes. Each note is a runnable workflow with concrete commands, expected outputs, and common variations.

A note belongs here if it would let an agent (or a colleague) reproduce a full simulation from a clean state — not a fragment of a workflow.

## Structure for a workflow note

```
1. Goal (one paragraph)
2. Prerequisites (env, mesh, ionic model, license, ...)
3. Commands (literal, copy-pasteable)
4. Expected outputs (file names, sizes, shapes)
5. Sanity checks (numerical values to confirm the run was correct)
6. Variations (common knobs to twist, with what they do)
7. Links (concepts that explain it, troubleshooting that catches its bugs)
```

## Current notes

(none yet — add as workflows are exercised end-to-end)

## Likely additions

- `single-cell-pacing.md` — bench pacing → limit cycle → reuse `.sv`. Cookbook from 01_basic_bench.
- `tissue-conduction.md` — generate slab mesh → tune conductivities → measure CV. Cookbook from 03A_study_prep_tuneCV.
- `apd-restitution.md` — bench `--restitute` workflow + slope analysis. Cookbook from 02B_APD_restitution.
- `parameter-sweep.md` — polling-file workflow. Cookbook from 02_EP_tissue/20_parameter_sweep.
- `reentry-induction.md` — S1-S2-S3 protocols (PEERP, RP_E, RP_B, PSD). Cookbook from 02_EP_tissue/21_reentry_induction.
- `em-coupling-prescribed.md` — prescribed λ(t) trace for EM-without-mechanics-solver studies. Cookbook from 06_EM_coupling.
- `em-coupling-fenics.md` — staggered openCARP + FEniCS-pulse. Cookbook to be developed when we go there.
- `lhs-pom.md` — Latin Hypercube Sampling Population of Models. Cookbook from 12_parallel_single_cell.
- `cellml-import.md` — CellML → EasyML → .so. Cookbook from 10_fromCellML.

Workflows should reference `findings/` for example numbers from past runs, and `troubleshooting/` for known pitfalls.
