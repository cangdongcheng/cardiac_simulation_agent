# tools/ep_simulation/openCARP/

openCARP is the open-source cardiac EP simulator. Multiple solver modes are exposed via the same binary, controlled by `.par` parameters.

| | |
| --- | --- |
| **Install** | `/usr/local/bin/openCARP` |
| **Source tree** | `/usr/local/lib/opencarp/share/openCARP/` |
| **Manual** | <https://opencarp.org/manual/opencarp-manual-latest.pdf> |
| **Tutorial source** | `/usr/local/lib/opencarp/share/tutorials/` |

## Solver modes

Each mode gets its own note when written. Currently:

| Mode | File | Status |
| --- | --- | --- |
| Monodomain | `monodomain.md` | not yet written |
| Bidomain | `bidomain.md` | not yet written |
| Eikonal | `eikonal.md` | not yet written |
| Reaction-eikonal | `reaction-eikonal.md` | not yet written |
| Pseudo-bidomain | `pseudo-bidomain.md` | not yet written |

When a tutorial under `tutorials/` exercises one of these modes specifically, lift the relevant generalizations into the corresponding mode note.

## Tutorials performed

The `tutorials/` subdirectory holds per-tutorial notes copied from the install tree. Each note is the raw record of what was set up, what ran, and what was found. Headline findings are summarized below.

### 01_EP_single_cell — single-cell experiments via `bench`

Index: [`tutorials/01_EP_single_cell/PROGRESS.md`](tutorials/01_EP_single_cell/PROGRESS.md)

Status legend: ✅ done · 📄 doc-only (no run.py to execute)

| Tutorial | Status | Headline finding |
| --- | --- | --- |
| [`01_basic_bench`](tutorials/01_EP_single_cell/01_basic_bench.md) | ✅ | TTP baseline APD90 304→289 ms over 20 s. Peri-infarct (GNa−62%, GCaL−69%, GKr−70%, GK1−80%) → peak Vm 39→20 mV, APD alternans ±11 ms, Cai peak 0.54→0.16 µM. |
| [`02B_APD_restitution`](tutorials/01_EP_single_cell/02B_APD_restitution.md) | ✅ | TTP S1S2 max-slope 1.23; Courtemanche 0.93 (atrial flatter); TTP Dynamic 1.31 (steeper). |
| [`03A_voltage_clamp`](tutorials/01_EP_single_cell/03A_voltage_clamp.md) | 📄 ✅ | Verified `bench --clamp -40 --clamp-dur 200 --clamp-start 10`. ICaL inward peak −0.88 µA/cm² at −40 mV. |
| [`04_limpet_fe`](tutorials/01_EP_single_cell/04_limpet_fe.md) | ✅ | `make_dynamic_model.sh my_MBRDR` works. APDshorten=1 → APD90=234 ms; APDshorten=3 → 128 ms. |
| [`05_EasyML`](tutorials/01_EP_single_cell/05_EasyML.md) | 📄 ✅ | Language reference. Markup: `.external/.nodal/.param/.trace/.lookup/.method`. Declarative; vars assigned once. |
| [`06_EM_coupling`](tutorials/01_EP_single_cell/06_EM_coupling.md) | ✅ | TTP+Niederer ~900 kPa/beat. TTP+Land17 cold start collapses to 0; recovers to 62 kPa over 10 s pacing — 20 s needed for limit cycle. |
| [`10_fromCellML`](tutorials/01_EP_single_cell/10_fromCellML.md) | 📄 ✅ | `cellml_converter.py` works on shipped LuoRudy94/Grandi/Severi. Modern converter auto-removes redundant α/β+diff. |
| [`11_toCellML`](tutorials/01_EP_single_cell/11_toCellML.md) | 📄 ✅ | EasyML → `.mmt` → `.cellml` via Myokit. Myokit not installed in opencarp env. |
| [`12_parallel_single_cell`](tutorials/01_EP_single_cell/12_parallel_single_cell.md) | ✅ | LHS PoMs over 10 Severi conductances. 5-sample mini-demo: HR 60↔120 bpm, Vm_max 5–29 mV. |

### 02_EP_tissue — tissue experiments via `openCARP`

Index: [`tutorials/02_EP_tissue/PROGRESS.md`](tutorials/02_EP_tissue/PROGRESS.md)

| Tutorial | Status | Headline finding |
| --- | --- | --- |
| [`00_simple`](tutorials/02_EP_tissue/00_simple.md) | ✅ | Bug: malformed Python `imp_region` override silently ignored; ran on `.par` defaults (DrouhardRoberge) instead of intended Courtemanche. |
| [`01_basic_usage`](tutorials/02_EP_tissue/01_basic_usage.md) | ✅ | 3 sub-exps: subthreshold ±20 → passive; suprathreshold +250 → propagating AP. CV ≈ 0.67 m/s. |
| [`02_stimulation`](tutorials/02_EP_tissue/02_stimulation.md) | ✅ | 6 stim modes verified. Bug: `extra_I_mono` doesn't switch to monodomain — `mod` list never appended to cmd. |
| [`03_study_prep_APD`](tutorials/02_EP_tissue/03_study_prep_APD.md) | ✅ | Baseline TTP APD80 = 291 ms @ BCL=700. GKs×2 → 242 ms; GCaL×1.5 → 316 ms. |
| [`03A_study_prep_tuneCV`](tutorials/02_EP_tissue/03A_study_prep_tuneCV.md) | ✅ | Default gi/ge → CV=0.638 m/s. Converges to 0.600 m/s in 2 iters. Tables for ar=1 and ar=3. |
| [`03B_study_prep_init`](tutorials/02_EP_tissue/03B_study_prep_init.md) | ✅ | 3 init methods: none, sv_init (uniform), prepace (spatially-varying by LAT). Ca_i differs substantially. |
| [`03C_tuning_wavelength`](tutorials/02_EP_tissue/03C_tuning_wavelength.md) | ✅ | Cable: GKs=0.22 → APD=342 ms (target 340). CVl=76, CVt=11 cm/s. Short-λ: λ_l=3.7 cm. |
| [`03D_conduction_velocity_restitution`](tutorials/02_EP_tissue/03D_conduction_velocity_restitution.md) | ✅ | CV restitution: min CI=325 ms default; −75% Gil → 300 ms; +50% Gil → 375 ms. |
| [`03E_study_resolution`](tutorials/02_EP_tissue/03E_study_resolution.md) | ✅ | Niederer benchmark: 100/200 µm agree within 0.5%; 500 µm off by 43%. |
| [`03F_ERP_restitution`](tutorials/02_EP_tissue/03F_ERP_restitution.md) | ✅ | Courtemanche AF: ERP 130–160 ms (S1=200–500 ms). Flat restitution. APD94 ≈ ERP−7 ms. |
| [`04_tagging`](tutorials/02_EP_tissue/04_tagging.md) | ✅ | Mesher (cm, first-match) vs dynamic retagging (µm, highest-wins). `-experiment 3` for model-build-only. |
| [`05A_Regions_vs_Gradients`](tutorials/02_EP_tissue/05A_Regions_vs_Gradients.md) | ✅ | MahajanShiferaw; region-vs-gradient APD diff <4 ms; GKr/GKs 0.1×–2.0× → APD 200–154 ms. |
| [`05B_Conductive_Heterogeneity`](tutorials/02_EP_tissue/05B_Conductive_Heterogeneity.md) | ✅ | 4 regions giL=2/0.5/0.125/0.03125 → CV=99/78/50/27 cm/s. CV ∝ √giL doesn't hold. |
| [`05C_Cellular_Dynamics_Heterogeneity`](tutorials/02_EP_tissue/05C_Cellular_Dynamics_Heterogeneity.md) | ✅ | GNa ×8 → CV 148 cm/s; GNa /8 → 43 cm/s. GNa modulates CV less than conductivity. |
| [`05D_Region_Reunification`](tutorials/02_EP_tissue/05D_Region_Reunification.md) | ✅ | `gvec[]` interface: LuoRudy94/Ramirez/tenTusscher/MahajanShiferaw. TTP uses uppercase M/H/J. |
| [`05E_Smooth_Gradient_Heterogeneities`](tutorials/02_EP_tissue/05E_Smooth_Gradient_Heterogeneities.md) | ✅ | TTP; GKr 0×–5× → APD 269–232 ms; Gto gradient has minimal APD effect. |
| [`07_extracellular`](tutorials/02_EP_tissue/07_extracellular.md) | ✅ | 3 methods: φ_e recovery, pseudo_bidomain, bidomain. pseudo_bidomain best trade-off. |
| [`07B_periodic`](tutorials/02_EP_tissue/07B_periodic.md) | ✅ | Patched (`stim_box` helper replaces `model.Stimulus`). Yper reentry induction with Vm-clamp block reproduces tutorial result. |
| [`08_lats`](tutorials/02_EP_tissue/08_lats.md) | ✅ | ACT 1–22 ms (mean 12.7), REP 239–254 ms (mean 246.6), APD 231–238 ms (mean 233.9). |
| [`13_laplace`](tutorials/02_EP_tissue/13_laplace.md) | ✅ | `-experiment 2` Laplace-only solve. φ_e normalized 0→1 left→right. 20,402 nodes, <5 s. |
| [`16_bidm_dogbone`](tutorials/02_EP_tissue/16_bidm_dogbone.md) | ✅ | Dogbone pattern confirmed under unequal anisotropy (g_il/g_it=9.2 vs g_el/g_et=2.65). |
| [`20_parameter_sweep`](tutorials/02_EP_tissue/20_parameter_sweep.md) | ✅ | 5-point sweep on `stim[0].start`, `stim[1].start`. Two-step workflow: generate poll file then run. |
| [`21_reentry_induction`](tutorials/02_EP_tissue/21_reentry_induction.md) | ✅ | PEERP induced sustained reentry with 2 ectopic beats (ERP=2201→2349 ms); RP_E(BCL=130) no reentry on --np 1. |

## Cross-cutting topics that may eventually need their own notes

- `parameter-dictionary.md` — how to use `carphelp` to discover flags; `--dry`-and-grep workflow
- `mesh-and-tags.md` — element tag conventions (1=tissue, 1234/1235=periodic links)
- `output-formats.md` — `.igb`, `.dat`, state-vector files, `parameters.par` as ground truth
- `bugs-catalog.md` — consolidated bug list across the tutorial sweep (8+ bugs documented in tutorial notes)

## On the relationship between this folder and the install

The notes under `tutorials/` are copies, not symlinks. Each note carries:
- The setup parameters (mesh, ionic model, BCs)
- The actual command run (with date)
- Run results, with units
- Lessons / transferable patterns

The original CLAUDE.md files remain in the install tree at `/usr/local/lib/opencarp/share/tutorials/<section>/<tutorial>/CLAUDE.md` for reference while running tutorials in place. Updates flow forward into this library when we revisit a tutorial; the install copies are the live record, the library copies are the curated record.

Most tutorial notes pre-date the library's frontmatter spec (see `../../../CLAUDE.md`). Bringing them into compliance is incremental — done when we revisit each note.
