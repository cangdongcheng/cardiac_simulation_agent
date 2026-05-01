# Tissue tutorials — exploration progress

Top-level index of the per-tutorial knowledge base. Each completed entry has its own `CLAUDE.md` inside that subdir with parsed setup, run results, and findings.

Status legend: ✅ done · ⚙️ running · ⏭️ skipped (with reason) · ❌ failed · ⏳ planned

| Tutorial | Status | Key finding / note |
| --- | --- | --- |
| `00_simple` | ✅ | Bug: malformed Python imp_region override silently ignored; sim ran on `.par` defaults (`DrouhardRoberge`) instead of intended `Courtemanche`. |
| `01_basic_usage` | ✅ | 3 sub-exps verified: subthreshold ±20 → passive response, suprathreshold +250 → propagating AP. CV ≈ 0.67 m/s. |
| `02_stimulation` | ✅ | 6 stim modes verified. Bug: `extra_I_mono` doesn't actually switch to monodomain — `mod` list never appended to `cmd`. |
| `03_study_prep_APD` | ✅ | Baseline tenTusscherPanfilov APD80 = 291 ms @ BCL=700. GKs*2 → 242 ms (−49 ms). GCaL*1.5 → 316 ms (+26 ms). |
| `03A_study_prep_tuneCV` | ✅ | Default gi/ge → CV=0.638 m/s. Converge to 0.600 m/s in 2 iters (gi=0.1534, ge=0.5510). Equal aniso (ar=1): gi_s=0.0367, ge_s=0.1317. Unequal (ar=3): gi_s=0.0313, ge_s=0.3376. Bug: `tuneCV` needs opencarp Python in PATH. |
| `03B_study_prep_init` | ✅ | 3 init methods: none (Vm=-86.93), sv_init (Vm=-85.71, uniform), prepace (Vm=-85.67±0.04, spatially varying by LAT). Ca_i differs substantially; init matters for long simulations. |
| `03C_tuning_wavelength` | ✅ | Cable: default GKs→APD=297 ms; GKs=0.22→APD=342 ms (target 340). CVl=76 cm/s, CVt=11 cm/s. Short-λ: GKs=0.25+Gil=0.02+Git=0.0135 → λ_l=3.7 cm. Sheet not run (session gap). |
| `03D_conduction_velocity_restitution` | ✅ | CV restitution: min CI=325ms default; −75% Gil→300ms; +50% Gil→375ms. Bug: `restituteCV` missing 8 parser args (fixed). |
| `03E_study_resolution` | ✅ | Niederer benchmark: 100/200µm agree within 0.5% (40.65 vs 40.87 ms at far corner); 500µm off by 43% (58.3 ms). |
| `03F_ERP_restitution` | ✅ | Courtemanche AF: ERP 130–160ms (S1=200–500ms). Flat restitution curve. APD94≈ERP−7ms. Use giL=0.405, geL=1.45474 to skip tuneCV. |
| `04_tagging` | ✅ | Mesher (cm, first-match) vs dynamic retagging (µm, highest-wins); `-experiment 3` for model-build-only. |
| `05A_Regions_vs_Gradients` | ✅ | MahajanShiferaw; region vs gradient APD diff <4 ms; GKr/GKs 0.1x–2.0x → APD 200–154 ms. |
| `05B_Conductive_Heterogeneity` | ✅ | 4 regions giL=2/0.5/0.125/0.03125 → CV=99/78/50/27 cm/s (isolated); CV ∝ √giL doesn't hold. |
| `05C_Cellular_Dynamics_Heterogeneity` | ✅ | GNa up x8 → CV 148 cm/s; GNa down x8 → 43 cm/s. GNa modulates CV less than conductivity. |
| `05D_Region_Reunification` | ✅ | gvec[] interface: LuoRudy94/Ramirez/tenTusscher/MahajanShiferaw; TTP uses uppercase M/H/J. |
| `05E_Smooth_Gradient_Heterogeneities` | ✅ | tenTusscherPanfilov; GKr 0x–5x → APD 269–232 ms; Gto gradient has minimal APD effect. |
| `07_extracellular` | ✅ | 3 methods: φ_e recovery (monodomain), pseudo_bidomain, bidomain. pseudo_bidomain best trade-off. Bath size → convergence with bidomain. |
| `07B_periodic` | ✅ | Patched (`stim_box` helper replaces `model.Stimulus`). Exp1 (noflux, 300 ms): planar S1 wave annihilates at top edge, ~3 m 20 s. Exp2 (Yper, 500 ms, block-Vm=-80, block-dur=90): 201 Y-periodic link elements; Vm clamp blocks Y re-entry for 90 ms then lifts — consistent with tutorial reentry induction protocol, ~6 m 3 s. |
| `08_lats` | ✅ | ACT 1–22 ms (mean 12.7), REP 239–254 ms (mean 246.6), APD 231–238 ms (mean 233.9). DrouhardRoberge bidomain, -10/-70 mV thresholds. |
| `13_laplace` | ✅ | `-experiment 2` Laplace-only solve. φ_e normalized 0→1 left→right. 20,402 nodes, <5 s. Used for fiber coordinate computation. |
| `16_bidm_dogbone` | ✅ | Dogbone pattern confirmed under unequal anisotropy (g_il/g_it=9.2 vs g_el/g_et=2.65). 255 s at 0.5 mm, --np 1. |
| `20_parameter_sweep` | ✅ | 5-point sweep: stim[0].start 20–60 ms, stim[1].start 50–70 ms. Two-step: generate poll file then run. Auto-subdirs per parameter set. |
| `21_reentry_induction` | ✅ | PEERP induced sustained reentry with 2 ectopic beats (ERP=2201→2349 ms); RP_E(BCL=130) no reentry detected on --np 1. Bug: both RP_E runs share jobID, second crashes with EOFError on overwrite prompt in batch mode. |

## Recurring observations

(Populated as the work progresses.)

## Recurring observations

- **mpiexec unavailable** on this system: all tissue tutorials run with `--np 1`. Tutorial notes recommending `--np >2` do not apply.
- **PATH must include opencarp env**: `PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH` needed for subprocess tools (tuneCV, restituteCV, meshtool).
- **numpy.loadtxt over carputils txt.read**: standalone analysis scripts must use `numpy.loadtxt(fname, skiprows=1)` — `carputils.carpio.txt.read()` requires the carputils decorator context.
- **CV formula**: `CV (cm/s) = 1000 / t_ms` where t is travel time in ms over 10 mm distance.

## Bugs catalog

1. **`00_simple/run.py:132-136`** — malformed Python override of `imp_region[0].im` (missing comma + missing `-` prefix). openCARP silently uses `.par` defaults.
2. **`02_stimulation/run.py:658`** — `mod` list (containing `-bidomain 0` for `extra_I_mono`) is never appended to the command. Mode silently runs as bidomain.
3. **`07B_periodic/run.py:211–218`** — [FIXED] `model.Stimulus` called with legacy `stimulus[]` geometry kwargs (`x0/y0/xd/yd`). These exist in openCARP as `stimulus[]` (legacy, confirmed via `carphelp`), but `model.Stimulus` generates `stim[]` params and rejects them at the Python layer. Fix applied: replaced all 4 `model.Stimulus` calls with a local `stim_box` helper that emits raw `-stim[N].elec.p0[i]`/`elec.p1[i]` indexed scalar flags + renamed `strength`→`pulse.strength`, `duration`→`ptcl.duration`, `start`→`ptcl.start`, `stimtype`→`crct.type`. Tutorial now runs successfully.
4. **`21_reentry_induction/run_remaining.sh` pattern** — both RP_E runs (end_bcl=130 and end_bcl=140) resolve to the same jobID `YYYY-MM-DD_RP_E`. In batch/piped mode, carputils's `overwrite_mode_prompt()` fails with `EOFError: EOF when reading a line` because stdin is closed. Fix: pass `--overwrite-behaviour overwrite` or use `--ID` to distinguish runs.
