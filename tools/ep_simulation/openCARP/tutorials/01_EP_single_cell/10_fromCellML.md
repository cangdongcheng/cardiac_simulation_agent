# CLAUDE.md — 01_EP_single_cell / 10_fromCellML

## Status: documentation-only

`run.py` is a parser stub. The tutorial walks through 3 worked CellML→EasyML conversions:

| | input | shipped final | difficulty | issues encountered |
| --- | --- | --- | --- | --- |
| Example 1 | `LuoRudy94.cellml` | `LuoRudy94.final.model` | basic | stimulus current named `I_st`; redundant α/β alongside τ/inf for `d`,`f` gates |
| Example 2 | `Grandi.cellml` | `Grandi.good.model` | hard | `V_m`/`I_app` non-default names; stiff Ca handling needs `cvode`; Markov needs `markov_be`; lookup-table singularity at V=−5 mV |
| Example 3 | `Severi.cellml` | `Severi.model` | hardest | seconds-not-ms units throughout; `pi` clashes with diff-var; `Kc` typo; `i_tot` units off; sinoatrial — runs without stimulus current |

## Available locally

- `cellml_converter.py`: `/usr/local/bin/cellml_converter.py` (also under `/usr/local/lib/opencarp/...`)
- `limpet_fe.py`: `/usr/local/bin/limpet_fe.py`
- `make_dynamic_model.sh`: `/usr/local/bin/make_dynamic_model.sh`

All three are on PATH inside the `opencarp` conda env.

## What `cellml_converter.py` does (verified 2026-04-28)

```bash
cellml_converter.py LuoRudy94.cellml > LuoRudy94.model
```

The shipped converter is **smarter than the tutorial implies** — it now auto-removes redundant `diff_X` when `alpha`/`beta` (or `tau`/`inf`) are also present, printing `ATTN:` lines. Output for LuoRudy94.cellml:

```
WARNING: stimulus current not found
ATTN: For m in fast_sodium_current_m_gate found diff+alpha+beta → Removing diff
ATTN: For h …  Removing diff
…
ATTN: For d in L_type_Ca_channel_d_gate found diff+alpha+beta+tau+inf → Removing diff
```

The tutorial's "Step 3: Open up LuoRudy94.model and manually delete the redundant diff equations" is no longer needed.

For Grandi:

```bash
cellml_converter.py --Vm=V_m --Istim=I_app Grandi.cellml > Grandi.model
```

Without `--Vm/--Istim` you get `WARNING: transmembrane voltage not found / stimulus current not found`. Inspect the .cellml manually (or grep for `voltage` and `stim`) to find the right names.

## Conversion workflow

```
.cellml ──cellml_converter.py──▶ .model ──limpet_fe.py──▶ .cc + .h
                                                      │
                              make_dynamic_model.sh ──┴──▶ .so ◀── bench --load-module
```

```bash
# 1. CellML → EasyML
cellml_converter.py [--Vm=NAME] [--Istim=NAME] [--out=FILE] [--trace-all] [--plugin] \
                    [--params] [--keep-redundant] foo.cellml > foo.model

# 2. EasyML → C++
limpet_fe.py foo.model /usr/local/lib/opencarp/share/openCARP/physics/limpet/models/imp_list.txt \
             /usr/local/lib/opencarp/share/openCARP/physics/limpet/src/imps_src

# 3. Compile to .so
make_dynamic_model.sh foo

# 4. Load and run
bench --load-module ./foo.so --duration 1000 --bcl 500 --stim-curr 30 --validate
```

## Common pitfalls (tutorial-derived)

1. **Stimulus current variable** — different authors use `I_st`, `i_Stim`, `I_app`, etc. If you don't pass `--Istim`, the converter leaves the variable in place and `limpet_fe.py` errors with `KeyError: Var("I_st")`. Either delete the line manually after conversion, or specify `--Istim`.

2. **Vm variable** — `V`, `V_m`, `V_ode`, `Vm`. Same fix: pass `--Vm=<name>` to the converter.

3. **Stiff Ca dynamics** — explosive Ca release during the upstroke can break forward-Euler. Symptom: NaNs at random t during long runs. Group Ca-related state vars under `.method(cvode)`:

   ```c
   group {
     Ca_i;  Ca_j;  Ca_sl;  Ca_sr;
     f_Ca_Bj;  f_Ca_Bsl;
   } .method(cvode);
   ```

4. **Markov submodels** — RyR / IKr Markov chains need `.method(markov_be)`:

   ```c
   group { Ry_Ri; Ry_Ro; Ry_Rr; } .method(markov_be);
   ```

5. **Lookup table singularities** — adding `V_m; .external(Vm); .nodal(); .lookup(-100,100,0.05);` may trip a singularity in some kinetic functions (e.g. tau_d at V=−5 in Grandi). Symptom: `LUT WARNING: V_m=-5 produces -nan in entry number N`. Fix with L'Hopital at the singular point:

   ```c
   tau_d = (V_m == -5.) ? (-d_inf/6./0.035)
                        : ((1.*d_inf*(1. - exp((-(V_m+5.)/6.))))/(0.035*(V_m+5.)));
   ```

6. **Unit mismatches (especially seconds vs ms)** — older sinoatrial-node CellML files (Severi 2012) use SI seconds. openCARP wants ms. Manually convert all time-based constants in the .model. Watch out for `F` in C/mol where you might need to multiply by 1000 for the right kinetics.

7. **Variable name conflicts** — EasyML demands global uniqueness. Some CellML names like `time`, `pi`, `do` collide with internal names; rename to `t`, `PI`, `var_do` etc.

8. **Sinoatrial cells (Severi)** — auto-pacing cells need `--stim-curr 0` because they fire spontaneously. With a forced stim, an extra AP appears immediately after t=0.

## Lessons (transferable)

1. **Conversion is semi-automatic** — `cellml_converter.py` handles 80% of the work, but every model needs hand-tweaks (units, integration methods, lookup tables). Plan for an iterative compile+test loop.
2. **Always start with the shipped fixed-up model** for the worked examples — `Grandi.good.model` and `Severi.model` reflect the necessary fixes; the raw `Grandi.model`/output of cellml_converter does not.
3. **Inspect `imp_list.txt`** at `/usr/local/lib/opencarp/share/openCARP/physics/limpet/models/imp_list.txt` to see what's already integrated. Adding a new model to the list and rebuilding via the openCARP CMake `-DUPDATE_IMPLIST=ON` makes the model available without dlopen.
4. **The tutorial assumes you have the openCARP source tree at a known path** (`/home/openCARP/...` in the docstring). Locally, the actual path is `/usr/local/lib/opencarp/share/openCARP/...`. The wrapped `make_dynamic_model.sh` and `limpet_fe.py` already know about this and don't need explicit paths.
