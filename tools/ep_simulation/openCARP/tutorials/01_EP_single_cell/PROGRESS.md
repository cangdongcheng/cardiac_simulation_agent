# Single-cell tutorials — exploration progress

Top-level index of the per-tutorial knowledge base. Each completed entry has its own `CLAUDE.md` inside that subdir with parsed setup, run results, and findings.

Status legend: ✅ done · ⚙️ running · ⏭️ skipped (with reason) · ❌ failed · ⏳ planned · 📄 doc-only (no run.py to execute)

| Tutorial | Status | Key finding / note |
| --- | --- | --- |
| `01_basic_bench` | ✅ | TTP baseline (exp01/02 reference): APD90 304→289 ms over 20 s. Peri-infarct (exp03/04): peak Vm 39→20 mV, APD alternans ±11 ms after 20 s, Cai peak collapse 0.54→0.16 µM. |
| `02B_APD_restitution` | ✅ | TTP S1S2 max-slope=1.23; Courtemanche=0.93 (flatter atrial); TTP Dynamic=1.31 (steeper). Tutorial's "slope>1 vs slope<1" classification not reproducible with only GKr/GKs (Table-2 recipe also needs G_pCa, G_pK, G_tf). Bug: jobID drops args.params, runs collide. |
| `03A_voltage_clamp` | 📄 ✅ | Doc-only stub. Verified `bench --imp TTP --clamp -40 --clamp-dur 200 --clamp-start 10` works directly. ICaL inward peak −0.88 µA/cm² during clamp at −40 mV; cell fires AP after release. |
| `04_limpet_fe` | ✅ | `make_dynamic_model.sh my_MBRDR` works on this system. APDshorten=1→APD90=234 ms; APDshorten=3→128 ms. Bug: run.py default --imp-par='APDshorten=3' silently overrides .model's default of 1. |
| `05_EasyML` | 📄 ✅ | Doc-only language reference. Key markup: .external/.nodal/.param/.trace/.lookup/.method. Declarative — assignments order-independent, vars assigned once. Self-reference traps via `if`. |
| `06_EM_coupling` | ✅ | TTP+Niederer: Tension ~900 kPa/beat. TTP+Land17 cold start collapses to 0; recovers to 62 kPa over 10 s pacing (20 beats) — 20 s needed for limit cycle. Bug: `update_niederer_stress_par` undefined (FIXED with stub). |
| `10_fromCellML` | 📄 ✅ | Verified `cellml_converter.py` works on shipped LuoRudy94/Grandi/Severi. Modern converter auto-removes redundant α/β+diff combos (tutorial's manual fix step is obsolete). |
| `11_toCellML` | 📄 ✅ | Doc-only. EasyML→.mmt→.cellml via Myokit. Myokit not installed in opencarp env (need `pip install myokit`). EasyML2mmt.py present. |
| `12_parallel_single_cell` | ✅ | LHS+multiprocessing PoMs over 10 Severi conductances. Mini-demo (5 samples, 10 s ea., seed=42): HR varies 60↔120 bpm, Vm_max 5–29 mV. Real-time factor ~3.5 → full 200×250s ≈ 30 min on 8 cores. |

## Recurring observations

- **Binary output (`--bin`)**: bench writes individual `<EP>.<sv>.bin` files of float64 doubles, one per state variable. Load via `np.fromfile(path, dtype=np.float64)`. Length = `duration / dt-out + 1`. Without `--bin`, all currents go to a single ASCII `Trace_0.dat` keyed by `<EP>_trace_header.txt`.
- **Limit cycle convergence**: 20+ s of pacing is required for human ventricular models (TTP, Grandi, ToR-ORd) at BCL=500. Cold-start (1 cycle) tension/Ca values are misleading; always save `.sv` and reuse with `--read-ini-file` or `--init`.
- **Per-tutorial parser collisions**: `tools.standard_parser()` adds `--stress-models` and `--ionic-models` (`store_true` flags). Tutorials adding their own `--stress` arg work fine in their own dirs (exact match wins) but the same flag prefix-collides outside (e.g. `--stress-models` interpreted instead of `--stress=value`). Always run from the tutorial's own directory.
- **`--imp-par` parameter modifier syntax**: `name[+|-|/|*|=][+|-]N[%]`. bench echoes the parsed modifiers at startup — always check this line to confirm.
- **`bench` real-time factor** (this machine, x86_64, default Intel HD Graphics 4400):
  - TTP forward Euler dt=0.01: ~12× (1 ms simulated per 0.083 ms wall)
  - my_MBRDR (Drouhard-Roberge): ~50× (smaller state vector)
  - Severi: ~3.5× (auto-paced, more equations)
  - Grandi+Land17 (strongly coupled): ~1.4× (largest state, plus stress plug-in)
- **Doc-only tutorials** (`03A`, `05`, `10`, `11`) ship `parser()` stubs but no `run()` body. Their RST docstring IS the tutorial; exercise the underlying `bench`/`limpet_fe.py`/`cellml_converter.py` directly per the docstring.

## Bugs catalog

1. **`02B_APD_restitution/run.py:319-324`** — `jobID` ignores `args.params` due to a missing `{}` placeholder: `'_{}'.format(args.imp,args.params)` interpolates only `args.imp`. Two runs with different `--params` collide on the same job ID, silently overwriting. Workaround: pass explicit `--ID`. Fix: `'_{}_{}'.format(args.imp, args.params.replace(',', '_').replace('=', ''))`.
2. **`04_limpet_fe/run.py:285-288`** — default `--imp-par='APDshorten=3'` silently overrides the model's intrinsic `.param() { APDshorten = 1; }` default. To get baseline behaviour you must explicitly pass `--imp-par 'APDshorten=1'`. Fix: change default to `''` or `None`.
3. **`06_EM_coupling/run.py:994` (FIXED)** — `setup_params` calls `update_niederer_stress_par()` which is not defined anywhere. Any `--stress stress_niederer` run crashes with `NameError`. Fix applied locally: added a stub function returning `{}`.
4. **`12_parallel_single_cell/run.py:237`** — `LatinHypercube(d=NUM_PARAMS)` not seeded. Each run produces different samples, so PoMs results are not reproducible without external seeding. Fix: pass `seed=...` to `LatinHypercube`.
5. **All single-cell run.py files** that use `--validate` and `--save-ini-file`: bench writes its outputs in cwd (not job.ID), then run.py does `shutil.move()` post-hoc to relocate them. Race condition if multiple instances of the same tutorial run concurrently from the same cwd.

