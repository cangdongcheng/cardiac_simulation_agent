# CLAUDE.md — 02_EP_tissue / 03A_study_prep_tuneCV

## What this tutorial demonstrates

Iterative **conduction velocity (CV) tuning**: given a target CV and a fixed ionic model + mesh resolution, use `tuneCV` to find the intracellular (`gi`) and extracellular (`ge`) conductivities that reproduce that CV. Also demonstrates how to compute conductivity pairs for orthotropic tissue (2D/3D) with prescribed **anisotropy ratios**.

Key concepts:
- CV ∝ √σ_m (monodomain conductivity). Only σ_m is free once ionic model, Δx, and time-stepping are fixed.
- σ_m = harmonic mean of σ_i and σ_e (monodomain ≡ bidomain when this relation holds).
- Iterative update: σ_{i,e}^{n+1} = σ_{i,e}^n × (CV_target / CV_measured)²
- For orthotropic tissue: run tuneCV separately for each direction (longitudinal, transverse, sheet-normal) to get three (gi, ge) pairs.
- Equal anisotropy ratio (α_lt = 1): intra and extra domains have same anisotropy; valid in both mono- and bidomain.
- Unequal anisotropy ratio (α_lt > 1): intracellular domain is more anisotropic; only representable in **bidomain**.

## Setup

| | value | source |
| --- | --- | --- |
| Tool | `tuneCV` (at `settings.execs.TUNECV`) | `run.py:582` |
| Default mesh resolution | 100 µm | `run.py:492` |
| Default gi / ge | 0.174 / 0.625 S/m | `run.py:498-501` |
| Default dt | 10 µs | `run.py:515` |
| Time-stepping | 0 (explicit Euler) | `run.py:510` |
| Cable length | 0.25 cm (shortened from default for speed) | `run.py:589` |
| Ionic model | `tenTusscherPanfilov` | `run.py:503` |
| Convergence tolerance | 0.0001 m/s | `run.py:588` |

`tuneCV` needs `#!/usr/bin/env python3` → must have opencarp Python first in PATH:
```bash
PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH \
  /home/cdc/miniconda3/envs/opencarp/bin/python run.py [args]
```

## CLI modes

| command | what it does |
| --- | --- |
| `./run.py` (no flags) | Measure CV with default gi/ge (no tuning) |
| `./run.py --velocity 0.6 --converge 1` | Iterate to find gi/ge that give CV=0.6 m/s |
| `./run.py --velocity 0.3 --converge 1` | Tune for transverse CV=0.3 m/s |
| `./run.py --ar 1.0` | Equal-anisotropy tuning: CVf=0.6, CVs=0.3, α_lt=1 |
| `./run.py --ar 3.0` | Unequal-anisotropy: CVf=0.6, CVs=0.3, α_lt=3 |
| `./run.py --compareTuning` | Compare CV vs resolution with/without tuning (3 resolutions, requires `--visualize` for plot) |
| `./run.py --compareTimeStepping` | Explicit Euler vs Crank–Nicolson |
| `./run.py --compareMassLumping` | With/without mass lumping |
| `./run.py --compareModelPar` | Full GNa vs GNa*0.5 |

Note: `--compareX` flags run 6 tuneCV instances (3 resolutions × 2 variants); each takes ~30 s, total ~3 min.

## Findings (run on 2026-04-28)

### Baseline measurement (default gi/ge, no convergence)

```
CV = 0.6385 m/s  [gi=0.1740, ge=0.6250, gm=0.1361]
```

Default conductivities yield CV≈0.64 m/s with tenTusscherPanfilov at 100 µm — slightly above the typical physiological target of 0.6 m/s.

### Convergence run: `--velocity 0.6 --converge 1`

```
Iter 0 (initial):  CV=0.6385 m/s [gi=0.1740, ge=0.6250, gm=0.1361]
Iter 1:            CV=0.6005 m/s [gi=0.1537, ge=0.5519, gm=0.1202]
Iter 2 (converged):CV=0.6000 m/s [gi=0.1534, ge=0.5510, gm=0.1200]
```

Converged in **2 iterations** because the initial guess was already close (factor = (0.6/0.6385)² ≈ 0.882 → one large correction, then fine-tuning). Final longitudinal conductivities: **gi=0.1534 S/m, ge=0.5510 S/m**.

### Anisotropy ratio: `--ar 1.0` (equal; CVf=0.6, CVs=0.3)

```
g_if: 0.153400   g_is: 0.036650   g_ef: 0.550990   g_es: 0.131660
```

Check: α_if = gi_f/gi_s = 0.1534/0.03665 ≈ 4.19. α_ef = ge_f/ge_s = 0.5510/0.13166 ≈ 4.19. Ratio α_lt = 1.0 ✓ (equal anisotropy enforced). With these conductivities, both mono- and bidomain produce the same CV.

### Anisotropy ratio: `--ar 3.0` (unequal; CVf=0.6, CVs=0.3)

```
g_if: 0.153400   g_is: 0.031330   g_ef: 0.550990   g_es: 0.337630
```

Check: α_lt = (gi_f × ge_s)/(ge_f × gi_s) = (0.1534 × 0.33763)/(0.5510 × 0.03133) ≈ 3.0 ✓. Note that ge_s is *larger* for ar=3 (0.338) than for ar=1 (0.132) — the extracellular transverse conductivity is raised to maintain CV while reducing the transverse intra/extra anisotropy imbalance. This parameter set can only be used in **bidomain** simulations; monodomain would compute incorrect virtual electrode patterns.

## Lessons (transferable)

1. **Always prepend opencarp env to PATH when running tuneCV** — it uses `#!/usr/bin/env python3` so the subprocess inherits `PATH`, not the calling Python. Fix: `PATH=/home/cdc/miniconda3/envs/opencarp/bin:$PATH python run.py ...`
2. **CV tuning is mesh- and model-specific.** A conductivity calibrated at 100 µm will be wrong at 200 µm. Run tuneCV for every (model, resolution) pair you want to use in production.
3. **`results.dat`** is the canonical output of a tuneCV run: `CV gi ge gm` on a single line. It's read by `txt.read(cvfile)` and overwritten each run — save it if you want to compare.
4. **Anisotropy ratio α_lt**: equal (=1) means mono/bidomain agree on CV patterns; unequal (>1) produces virtual electrodes visible only in bidomain. For pure propagation studies the difference is small, but for defibrillation/pacing studies it matters.
5. **Initial guess matters for convergence speed.** Starting near the target (default gi/ge → CV≈0.64 for target 0.6) converges in 2 iterations; targets far from default (CV=0.3) may need 3–4 iterations (see tutorial docstring examples).
6. **`table.dat`** is written alongside `results.dat` and contains the per-iteration history (useful for diagnosing slow convergence or oscillation).
