# CLAUDE.md — 01_EP_single_cell / 11_toCellML

## Status: documentation-only

`run.py` is a parser stub. The shipped `convert.py` is a small Myokit wrapper. The tutorial walks through EasyML → CellML conversion using Myokit's `.mmt` intermediate format.

## Conversion pipeline

```
.model (EasyML) ──EasyML2mmt.py──▶ .mmt (Myokit) ──convert.py──▶ .cellml
```

```bash
# Step 1: EasyML → .mmt
python3 /usr/local/lib/opencarp/share/openCARP/physics/limpet/src/python/EasyML2mmt.py \
        DrouhardRoberge.model

# Step 2: .mmt → .cellml (uses Myokit's CellML writer)
python3 convert.py DrouhardRoberge   # reads DrouhardRoberge.mmt, writes DrouhardRoberge.cellml
```

## EasyML2mmt.py options (from --help)

| flag | purpose |
| --- | --- |
| `--verbose` | print conversion details |
| `--init_component CNAME` | initial component name (default: `cell`) |
| `--istim NAME` | name to use for the stimulus current (default: `I_stim`) |
| `--pace_name NAME` | name for the pacing variable (default: `pace`) |
| `--tend FLOAT` | simulation duration written into the .mmt protocol (default: 1100 ms) |
| `--bcl FLOAT` | pacing cycle length in the .mmt protocol (default: 500 ms) |
| `--plugin` | mark the model as a plug-in (modifies `Iion +=` rather than `Iion =`) |

## Component conventions

EasyML has no native concept of CellML "components" — the converter introduces them via comments:

```c
# COMPONENT my_section
GNa = 15;
g_Na_init = 0.05;
```

or `// COMPONENT my_section`. All variables and equations belong to the *last* component declared. Default component is `cell`. Useful for organizing Markov chains or independent ion-channel groups when round-tripping to CellML.

## Local availability

- `EasyML2mmt.py`: `/usr/local/lib/opencarp/share/openCARP/physics/limpet/src/python/EasyML2mmt.py` — present.
- `convert.py`: in this tutorial directory. Reads from `<name>.mmt` and writes `<name>.cellml`.
- **Myokit**: NOT installed in the `opencarp` conda env (`ModuleNotFoundError: No module named 'myokit'`). To run this tutorial:

  ```bash
  /home/cdc/miniconda3/envs/opencarp/bin/pip install myokit
  ```

  After install, `convert.py` can be run.

## Known limitations (from the docstring)

- **Groups within groups not supported** — flatten any nested `group { group { ... } ... }` blocks before conversion.
- **Hyperbolic trig functions not supported** — replace `sinh`, `cosh`, `tanh` with their exponential equivalents in the .mmt or pre-process the .model.
- **Reserved names**: avoid using `d` or `inf` as variable names (collides with differential-variable annotations / infinity-state notation).

## Why this matters

CellML is the publishing format. To upload your custom-developed openCARP/EasyML model to the `models.cellml.org` repository for community sharing, you need a CellML version. The .mmt step is just a convenience format Myokit understands.

## Lessons (transferable)

1. **Round-tripping is non-trivial** — EasyML→Myokit→CellML is lossy in both directions. Hand-curated tweaks (lookup tables, custom integration methods, `.flag` markup) don't survive the round-trip — they're EasyML-specific extensions.
2. **Myokit is the key dependency** — add `myokit` to your conda env if you plan to publish models. `pip install myokit` is sufficient.
3. **CellML metadata** — once you have the .cellml file, manually add metadata blocks (author, paper reference, units, parameter ranges) before uploading. The auto-generated file has none.
4. **Companion to 10_fromCellML** — the two tutorials are bidirectional. The fact that a model came from CellML doesn't mean it round-trips cleanly back: the `.method(cvode)` you added in step 4 of fromCellML is lost on the way back.
