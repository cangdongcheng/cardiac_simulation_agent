# CLAUDE.md — 02_EP_tissue / 00_simple

## What this tutorial demonstrates

A minimal end-to-end carputils tissue example: programmatic mesh generation (`mesh.Block`), conductivity setup (`gregion`), ionic model assignment, S1 stimulus on the −x face, monodomain electrics, LAT post-processing. Authors tag it as "a template to base your own experiment on" — it's the simplest tissue tutorial in the tree.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | `mesh.Block(size=(2, 0.5, 0.5))` mm cuboid, fibres along x (0°/0°/90°/90°) | `run.py:92-95` |
| Solver type | monodomain (`bidomain=0`, set in `simple.par`) | `simple.par` |
| Ionic model | `DrouhardRoberge` (from `simple.par`) — *intended* `Courtemanche` from Python override never landed (see Bug below) | `simple.par`, `run.py:132-136` |
| Conductivity (region tag 1) | gₑₗ=0.625, gₑₜ=gₑₙ=0.236, gᵢₗ=0.174, gᵢₜ=gᵢₙ=0.019 S/m, scaled by `g_mult=0.5` | `run.py:117-129` |
| Stimulus | transmembrane current via `mesh.block_bc_opencarp` on the x=−1 face, 250 µA/cm², 2 ms at t=0 | `simple.par` + `run.py:150` |
| Output | spacedt=1 ms, timedt=1 ms; LATs computed via `lats[0]` | `simple.par`, `run.py:140-146` |
| Default `--tend` | 20 ms | `run.py:73-75` |

## Running

```bash
cd /usr/local/lib/opencarp/share/tutorials/02_EP_tissue/00_simple
/home/cdc/miniconda3/envs/opencarp/bin/python run.py --tend 20
# add --visualize to launch meshalyzer (requires display)
```

## Findings (run on 2026-04-28)

- Run completed cleanly, 20 ms simulation, np=1, ~0.9 s electrics compute time.
- Output dir: `2026-04-28_simple_20.0_petsc_np1/`. Contains `vm.igb`, `init_acts_activation-thresh.dat` (LATs), `parameters.par`, plus log files.
- AP elicited and propagated as expected (verified by non-zero LAT field).

### Bug: malformed Python override of ionic model

`run.py:132-136` is supposed to override the ionic model defined in `simple.par`. It contains two defects that combine to make the entire block a no-op:

```python
cmd += ['num_imp_regions',          1,                 # missing leading '-'
        'imp_region[0].im',         'Courtemanche'    # MISSING COMMA
        'imp_region[0].num_IDs',    1,                # → adjacent string-literal concat
        'imp_region[0].ID[0]',      1
]
```

Effect:
- The missing comma after `'Courtemanche'` makes Python concatenate `'Courtemanche' 'imp_region[0].num_IDs'` into a single string `'Courtemancheimp_region[0].num_IDs'`.
- All four flag names lack the `-` prefix that openCARP requires for command-line parameters.

openCARP silently treats the resulting tokens as unrecognized positional arguments and ignores them. The simulation runs using the `imp_region[0].im = DrouhardRoberge` value from `simple.par`. Verified by:

```bash
grep -E "imp_region\[0\]\.im\s*=" 2026-04-28_simple_20.0_petsc_np1/parameters.par
# → imp_region[0].im = DrouhardRoberge
```

The dry-run shows the malformed tokens trailing on the gregion line:
```
-gregion[0].g_mult 0.5 num_imp_regions 1 imp_region[0].im Courtemancheimp_region[0].num_IDs 1 imp_region[0].ID[0] 1
```

To fix: add commas and `-` prefixes:
```python
cmd += ['-num_imp_regions', 1,
        '-imp_region[0].im', 'Courtemanche',
        '-imp_region[0].num_IDs', 1,
        '-imp_region[0].ID[0]', 1]
```

But functionally the override is unnecessary because `simple.par` already defines an ionic model. The cleanest fix is to either delete the Python block or change `simple.par` to match the intended model. Keep the bug intact in this read-only tutorial; just be aware when copying this script as a template.

## Lessons (transferable)

1. **openCARP's CLI parser is permissive.** Unrecognized positional tokens are dropped silently — a successful simulation does not prove that all your flags were honored. Always verify via `<job>/parameters.par` (last-wins).
2. **Mixing `.par` files and Python flags works** because openCARP applies them in order with last-wins semantics, but it's easy to fool yourself when one or the other has a typo.
3. **`tools.simfile_path()` vs `tools.carp_path()`**: this script uses `simfile_path` for the `.par`. Both resolve a relative path, but `simfile_path` checks multiple search roots; safer for portable scripts.
4. **`mesh.block_bc_opencarp(geom, entity, index, coord, lower=True, bath=False)`** generates a face-stimulus electrode covering the lower (or upper) face of `coord`. The 5th positional arg is `lower`, not `upper`; `True` (this tutorial's call) → stim on the `-x` face, consistent with the docstring's claim that the stimulus sits on `x=−1.0`.
