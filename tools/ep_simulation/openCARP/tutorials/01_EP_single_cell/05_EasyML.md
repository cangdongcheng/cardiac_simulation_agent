# CLAUDE.md — 01_EP_single_cell / 05_EasyML

## Status: documentation-only

`run.py` is a parser stub — no executable content. The tutorial is a language reference for **EasyML**, the markup language used in `.model` files (consumed by `limpet_fe.py` to generate ionic-model C++ source). All practice runs happen in 04_limpet_fe.

## EasyML at a glance

EasyML is **declarative** — assignments can appear in any order; the translator builds a dependency tree. Key consequences:

1. Variables are assigned **once**. A variable that needs to depend on its own previous value (e.g. clamping) requires an intermediate name.
2. Looks like C with semicolons, but is **not Turing-complete** — no loops, no functions, no `#ifdef`. Use `arrayifier.py` for tabulated/compartmentalized expressions if needed.

### Three statement types

| Type | Form | Example |
| --- | --- | --- |
| Assignment | `var = expr;` | `Iion = INa + ICaL;` |
| Declaration (markup target) | `var;` | `Iion; .external();` |
| Markup | `.markup(args);` | `.lookup(-100, 100, 0.05);` |

### Declarative quirk to watch for

```c
A = 50*x*x + 40*x + 100;
if (A > 2000) { A = 2000; }   // ← TRAP
```

The `if` rewrites to `A = (A > 2000) ? 2000 : 50*x*x+40*x+100`, making `A` self-referential → A becomes a **state variable** silently. Always introduce an intermediate:

```c
A_factor = 50*x*x + 40*x + 100;
A        = A_factor;
if (A_factor > 2000) { A = 2000; }
```

## Recognized markup (high-traffic ones)

| Markup | Effect |
| --- | --- |
| `.external([name])` | Variable exposed to LIMPET (e.g. `V; .external(Vm);`) |
| `.nodal()` | Per-node value (vs per-region or scalar) |
| `.regional()` | Constant per region (e.g. fiber/sheet conductivity) |
| `.param()` | Tweakable from the command line via `--imp-par <name>=<v>` |
| `.trace()` | Becomes a column in the `Trace_*.dat` output |
| `.lookup(min,max,step)` | Build a lookup table — speeds up V-dependent expressions |
| `.lb()` / `.ub()` | Hard clamp lower/upper bound |
| `.units(expr)` | Physical units (used in code-gen sanity checks) |
| `.method(name)` | Integration method: `fe`, `rk2`, `rk4`, `rush_larsen`, `sundnes`, `markov_be`, `rosenbrock`, `cvode` |
| `.flag(A,B,…)` | Variable can only take values in this set, controlled by `flags` parameter |
| `.store()` | Manually update; ignore the differential equation (used for analytic Ca solutions, state machines) |
| `.array(N)` | Tile a variable into a length-N array (used with `arrayifier.py`) |

### Group syntax

Apply markup to many variables at once:

```c
group {
  GNa;
  Gsi;
  APDshorten = 1;
} .param();

group {
  Ca_i;
  Ca_j;
  Ca_sl;
} .method(cvode);
```

Groups can nest. Trace and parameter groups are how shipped models like `Grandi.model` declare which currents are visible and which conductances are tunable.

## Special variable names

| Name | Meaning |
| --- | --- |
| `t` | Absolute time (ms) |
| `dt` | Integration step (ms) — **avoid** in equations; use `diff_XXX` or `d_XXX_dt` instead |
| `diff_XXX` / `d_XXX_dt` | Time derivative of `XXX` (ms⁻¹) |
| `XXX_init` | Initial value of state variable `XXX` |
| `Iion` | Total ionic current (output of the model) |
| `Vm` | Transmembrane voltage (input/output via `.external()`) |

### Hodgkin-Huxley gates

A variable `XXX` becomes a gate automatically when *both*:

1. `XXX` appears in another equation (it's used somewhere), AND
2. Either `(alpha_XXX, beta_XXX)` or `(tau_XXX, XXX_inf)` is defined.

Translator then generates `diff_XXX` and uses Rush-Larsen integration. **Never** define both pairs and `diff_XXX` simultaneously — the translator either errors or picks one silently.

## Recommended units (SI baseline used by openCARP)

| Quantity | Unit |
| --- | --- |
| membrane current | µA/cm² (= pA/pF, given Cm) |
| dX/dt | ms⁻¹ |
| [Ca]ᵢ | µM |
| other [X] | mM |
| voltage | mV |
| conductance | mS/cm² |
| τ | ms |
| α, β | ms⁻¹ |

Internal units within your `.model` can be anything you like — but you must convert at the boundary (state-variable derivatives, `Iion`, externally-exposed concentrations). Mismatched units against `Vm` (mV) and `t` (ms) cause silent integration errors.

## Plug-in vs ionic model

| | Ionic model | Plug-in |
| --- | --- | --- |
| Sets `Iion` | `Iion = …` | `Iion = Iion + …` (additive) |
| Loaded via | `--imp=<name>` or `--load-module=<m>.so` | `--plug-in=<name>` |
| Markup hint | full model | `cellml_converter.py --plugin` flag |

Plug-ins layer extra channels (e.g. `IKATP_Ferrero`, `Stress_Land17`) on top of an existing ionic model.

## Lessons (transferable)

1. **EasyML is declarative** — don't think in loops or sequence; think in equations and dependencies.
2. **Markup is positional** — `.markup()` applies to the variable definition or declaration *immediately preceding* it (or to all members of an enclosing `group { ... }.markup();`).
3. **Lookup tables are the main perf knob** — `V; .lookup(-100, 100, 0.05);` makes every V-dependent function call a table lookup. Add lookup tables only after the model is verified correct, since tables can hide singularities (they'll show up as `LUT WARNING: produces -nan`).
4. **`.method(cvode)` for stiff Ca/Markov groups** — Markov submodels (`.method(markov_be)`) and explosive Ca release (`.method(cvode)`) are the two cases where forward-Euler / Rush-Larsen aren't enough. Group them and pick the integrator.
5. **`arrayifier.py`** unrolls array-syntax expressions (`V[1:-1] = ...`) into N scalar variables for compartmentalized models — handy for 1D cable diffusion or multi-compartment cells in single-cell mode.
