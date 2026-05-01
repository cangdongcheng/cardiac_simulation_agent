# troubleshooting/

The **FIX**: bug catalog and known-good patches. Things that broke, why, and the workaround.

A note belongs here if it documents a real failure mode (with reproducer + symptoms) and a verified fix.

## Structure for a troubleshooting note

```
1. Symptoms (literal error messages, observed misbehavior)
2. Root cause (what's actually wrong, with file:line if known)
3. Fix (commands, patch, config change)
4. Verification (how to confirm the fix worked)
5. Scope (which versions / setups are affected)
```

## Current notes

| File | Status | Symptom |
| --- | --- | --- |
| `meshalyzer-haswell.md` | validated | `ERROR::SHADER::VERTEX::COMPILATION_FAILED` and/or vertex picking does not select nodes — Intel Haswell or WSL |

## Likely additions (bugs already cataloged in tutorial PROGRESS.md files)

- `tutorial-bugs.md` — consolidated catalog of all bugs found across the tutorial sweep:
  - `02_EP_tissue/00_simple/run.py:132-136` — malformed `imp_region[0].im` override silently ignored
  - `02_EP_tissue/02_stimulation/run.py:658` — `mod` list never appended to cmd; `extra_I_mono` mode silently runs as bidomain
  - `02_EP_tissue/07B_periodic/run.py:211-218` — model.Stimulus rejects `x0/y0/xd/yd` legacy kwargs (FIXED with `stim_box` helper)
  - `02_EP_tissue/21_reentry_induction/run_remaining.sh` — RP_E runs collide on jobID, second crashes with EOFError
  - `01_EP_single_cell/02B_APD_restitution/run.py:319-324` — jobID drops `args.params`
  - `01_EP_single_cell/04_limpet_fe/run.py:285-288` — default `--imp-par='APDshorten=3'` silently overrides .model default
  - `01_EP_single_cell/06_EM_coupling/run.py:994` — `update_niederer_stress_par()` undefined (FIXED with stub)
  - `01_EP_single_cell/12_parallel_single_cell/run.py:237` — LatinHypercube not seeded
- `environment.md` — PATH/conda quirks: needing `opencarp` env on PATH for tuneCV/restituteCV/cellml_converter.py; numpy.loadtxt vs carputils.carpio.txt.read context dependency.
- `argparse-prefix-collisions.md` — `--stress` vs `--stress-models` and similar carputils prefix-matching pitfalls.
- `mpiexec-unavailable.md` — running with `--np 1` only; carputils tutorial recommendations of `--np >2` don't apply.

## Distinction from `findings/`

`findings/` documents what's *expected* (validated normal behavior). This directory documents what's *broken* and how to fix it.
