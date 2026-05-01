# CLAUDE.md — 01_EP_single_cell / 06_EM_coupling

## What this tutorial demonstrates

Couple a cellular EP model with an active-tension model and run pacing protocols on a single cell. Three coupling regimes:

1. **Activation-based (e.g. `stress_niederer`, "tanh")** — tension transient depends only on activation time and stretch; cheap, easy to fit, no Ca dependence.
2. **Weakly Ca-coupled (e.g. `stress_land12`, `stress_land17`)** — tension driven by cytosolic Ca²⁺ but EP not affected by mechanics.
3. **Strongly coupled (e.g. `--EP gpb_land`, `augustin`, `torord_landhumanstresswithpassive`)** — Land buffer replaces the EP model's TnC equation, so length/stretch feeds back into Ca handling and EP.

Optional stretch protocols can be prescribed via `--fs` (Lambda) and `--fsDot` (delLambda), or held isometric at `--stretch <λ>`.

## Setup

| | value | source |
| --- | --- | --- |
| Solver | `bench` | run.py:1152 (`settings.execs.BENCH`) |
| EP model choices | all `model.ionic.keys()` (40+ models) | run.py:786 |
| Stress choices | `stress_land12`, `stress_land17`, `stress_lumens`, `stress_niederer`, `stress_rice`, `none` | run.py:790 |
| Active default stim | 30 µA/cm² | run.py:1175 |
| Passive (no stim) | `--experiment passive` → stim_curr=0 | run.py:1172 |
| dt | 0.01 ms | run.py:1186 |
| dt-out | 0.1 ms | run.py:1196 |

## Bug 1 (FIXED): missing `update_niederer_stress_par()`

`run.py:994` calls `update_niederer_stress_par()` but the function isn't defined anywhere. Patched locally by adding a no-op stub returning `{}` (defaults take effect):

```python
def update_niederer_stress_par():
    """
    Default Niederer stress parameters. Returning {} keeps all plugin defaults
    in effect — added to fix NameError at run.py:994.
    """
    return {}
```

Without this fix, any run with `--stress stress_niederer` raises `NameError: name 'update_niederer_stress_par' is not defined`.

## Bug 2: argparse prefix collision with `--stress-models`

`tools.standard_parser()` adds `--stress-models` (a `store_true` flag for "print stress-models and exit"). The tutorial's own `--stress <name>` arg lives in the `experiment specific options` group. argparse handles `--stress` correctly when both are registered (exact-match wins over prefix), so `--stress stress_land17` works as expected **inside this tutorial directory**. The collision *does* fire in tutorials that don't define their own `--stress` (e.g. 04_limpet_fe), where `--stress` prefix-matches `--stress-models` and triggers the help dump. So the workaround is "use the right tutorial directory" — there's nothing to fix in 06.

## Output files

For coupling `<EP>_<stress>` (e.g. `tentusscherpanfilov_stress_land17`):

| File | Content |
| --- | --- |
| `<prefix>.t.bin` | Time vector |
| `<prefix>.Vm.bin` | Transmembrane voltage |
| `<prefix>.Iion.bin` | Total ionic current |
| `<prefix>.Tension.bin` | Active tension (kPa) |
| `<prefix>.Lambda.bin` | Fiber stretch (constant if no `--fs`) |
| `<prefix>.delLambda.bin` | Fiber stretch rate (constant 0 if no `--fsDot`) |
| `<expID>_<EP>_<stress>_bcl_<bcl>_ms_dur_<dur>_ms.sv` | End-of-run state vector |

## Findings: Run results (2026-04-28)

### exp01 — TTP + Niederer tanh, 2 sec, BCL=500

`./run.py --EP tentusscherpanfilov --stress stress_niederer --duration 2000 --bcl 500 --ID 2026-04-28_exp01_niederer`

- 4 beats over 2 sec; wall time ~0.8 s.
- Vm: rest −86.2 mV → peak 39.3 mV (TTP standard AP).
- **Tension peaks per beat (kPa): 893, 778, 760, 912.** Variability comes from Tref defaults; tutorial expects ~100 kPa peak after parameter tuning (`Tpeak=100, tau_c0=40, tau_r=110, t_dur=400`). The activation-based Niederer model fires every beat with the same shape, modulated only by stretch (here λ=1.0 ⇒ no length effect).
- Lambda constant at 1.000; delLambda at 0.0 (isometric).

### TTP + Land17 weakly coupled, 2 sec, BCL=500 (NOT yet at limit cycle)

`./run.py --EP tentusscherpanfilov --stress stress_land17 --duration 2000 --bcl 500 --ID 2026-04-28_exp_land17`

- Tension peaks (kPa): 3.1, 0.0, 0.0, 0.1.
- The Land model's force depends on the Ca transient and on TRPN (troponin) buffer state. From cold-start, the SR depletes after the first AP and tension collapses for 2-3 beats before reloading. Reaches steady-state only after many seconds of pacing.

### TTP + Land17, **10 sec** pacing, BCL=500 (approaching limit cycle)

`./run.py --EP tentusscherpanfilov --stress stress_land17 --duration 10000 --bcl 500 --ID 2026-04-28_exp_land17_long`

20 beats. Tension peak per beat (kPa):

```
3.1, 0.0, 0.0, 0.1, 0.2, 0.4, 0.6, 0.9, 1.4, 2.8,
5.2, 9.1, 14.5, 21.2, 28.7, 36.5, 44.1, 51.0, 57.2, 62.6
```

- Last 5 / first 5 ratio: 72×. **Still climbing at t=10 s** — confirms tutorial's recommendation to run 20 sec (40 beats) for limit-cycle convergence.
- Pattern: initial transient when Ca reservoir is depleting, then exponential rise as SR reloads and TRPN fraction climbs. Asymptotic value (extrapolating) ≈ 80-100 kPa, consistent with physiological human ventricular tension.

### Grandi + Land17, 2 sec, BCL=1000 (cold start)

`./run.py --EP grandi --stress stress_land17 --duration 2000 --bcl 1000 --ID 2026-04-28_exp_grandi_land17`

- Vm rest −81.8, peak +55.0 mV (Grandi has more positive overshoot than TTP).
- Tension peaks: 3.1 first beat, then 0.0 / 0.0 / 0.0 — same cold-start signature as TTP+Land17.
- Wall time ~7 sec — Grandi is significantly slower (real-time factor 1.4 vs 7.8 for TTP+Land17), reflecting Grandi's larger state vector.

## Conceptual summary

### Why limit-cycle runs are needed

- Activation-based stress (`stress_niederer`, tanh) produces consistent tension immediately — no calcium dependence.
- Calcium-driven stress (`stress_land12/17`, `stress_rice`) requires the Ca handling system to reach steady state. SR Ca content, TRPN occupancy, sodium and chloride concentrations all evolve over 20-60 seconds of pacing.
- **Always pace 20+ seconds** with these models, save the `.sv`, and reuse via `--init` for downstream experiments.

### Strongly vs weakly coupled

| Variant | Mechanism | Modeled by |
| --- | --- | --- |
| Weak | Stress is a function of Ca²⁺ and λ; EP is unaffected by mechanics | `--EP <ep> --stress stress_land17` |
| Strong | The EP model's TnC equation is replaced by the Land TRPN equation; mechanical state feeds back into Ca buffering | `--EP gpb_land`, `--EP augustin`, `--EP torord_landhumanstresswithpassive` (any name with `_land*` suffix) |

The tutorial detects strong coupling by suffix — see `STRONG_CIS = ("_landhuman", "_land", "_landhumanstress", "_landhumanstresswithpassive", "_rice")` at run.py:774.

### Stretch protocols

- `--stretch λ` (default 1.0) → constant strain held throughout the run.
- `--fs <file>` → trace-driven Lambda(t). File format: ASCII (t, λ) pairs.
- `--fsDot <file>` → trace-driven delLambda(t). Optional; if missing, length-dependence operates without explicit velocity.
- Bound to bench via `--clamp-SVs Lambda[:delLambda]` and `--SV-clamp-files <files>`.

## Lessons (transferable)

1. **Ca-driven active stress models always need 20+ s pacing** to reach a useful limit cycle. The first beat's tension is misleading. Save `.sv`, reuse with `--init`.
2. **`update_niederer_stress_par` bug** — missing function definition in this run.py. Fix is one-liner returning `{}`. Without it, ALL niederer experiments crash.
3. **Strong coupling = special EP names**: only EP models whose `model.ionic.keys()` entry ends with `_land*` or `_rice` are treated as strongly coupled. Naming convention is the API.
4. **Output file naming**: `<EP>_<stress>` for non-strong coupling, `<EP>_strongly_coupled` for strong. The job ID encodes this — useful for diff-and-compare across runs.
5. **Lambda/delLambda are state variables on the EP side**, not separate vars. They appear in the binary outputs even when no stretch protocol is applied (constant 1.0 / 0.0).
6. **Niederer defaults give tension peaks of ~900 kPa**, not the ~100 kPa expected physiologically — the tutorial's tanh figure expects parameter overrides (`Tpeak=100, tau_c0=40, ...`). Pass via `--stress-params 'Tpeak=100,tau_c0=40,tau_r=110'`.
7. **The activation-based vs Ca-driven distinction is qualitative**: activation-based gives identical tension shape on every beat once activated; Ca-driven shows beat-to-beat variation because Ca cycling is a slow process.
