---
title: Cardiac electromechanics — coupling models and tools
status: mixed   # math borrowed from papers (Land 2017, Holzapfel-Ogden, Niederer); tools landscape verified locally
use_when: question is about how Vm → Ca → tension → deformation are coupled, or which tool to choose for full EM
last_updated: 2026-05-01
---

# Cardiac electromechanics — coupling models and tools

A reference on the mathematical model and software landscape for full cardiac electromechanics (EM) coupling: voltage → calcium → active tension → tissue deformation, plus the geometric feedback from deformation back into the diffusion operator.

Equations are rendered in Unicode + monospace blocks for terminal-friendly viewing. For LaTeX-rendered output, paste the source into a Markdown viewer that supports MathJax.

---

## The three layers

Cardiac EM is canonically decomposed into three coupled subproblems with very different mathematical character:

```
   Layer 1                 Layer 2                Layer 3
   Vm dynamics       →     Active tension   →     Deformation
   ─────────────           ─────────────          ─────────────
   parabolic PDE           ODE system,            elliptic PDE,
   + ionic ODEs            algebraic              quasi-static
   ~0.01 ms steps          ~0.1 ms steps          ~1 ms steps
```

The forward chain (1 → 2 → 3) dominates physiologically. Two backward arrows exist:

- **Geometric coupling (always)**: deformation transforms the conductivity tensor in the deformed configuration.
- **Mechano-electric feedback (optional)**: stretch-activated channels and length-dependent Ca buffering.

---

## Layer 1 — Electrophysiology

### Monodomain reaction-diffusion equation

```
              ∂Vm
   β · Cm · ───── = ∇·(D ∇Vm) − β · I_ion(Vm, s) + I_stim
              ∂t
```

| symbol | meaning | typical value |
|---|---|---|
| Vm | transmembrane voltage | −86 mV rest, +40 mV peak |
| β | surface-to-volume ratio | ~1400 cm⁻¹ |
| Cm | membrane capacitance | 1 µF/cm² |
| D | conductivity tensor (orthotropic) | ~0.001 S/cm along fiber, ¼ across |
| I_ion | total ionic current | from ionic model |
| I_stim | applied stimulus current | external |
| s | state vector | 20-50 vars per node |

### Ionic model (ODE system)

For each Hodgkin-Huxley gate x:
```
   dx        x_∞(Vm) − x
   ─── = ─────────────────
   dt          τ_x(Vm)
```

For ion concentrations, e.g. cytosolic Ca:
```
   d[Ca]i              I_Ca
   ────── = − ──────────────────── + buffering + SR fluxes
     dt        2 · F · V_cyto
```

The full ionic model packages all gates and concentrations:
```
   ds/dt   = f(Vm, s)
   I_ion   = g(Vm, s)
```

Standard model families:

| Family | Examples | Notes |
|---|---|---|
| Hodgkin-Huxley | TTP, OHara-Rudy, ToR-ORd (ventricle); Courtemanche, Maleckar (atria); Severi, Fabbri (SAN) | most common |
| Markov state models | INa Bondarenko, IKr Markov | for channels where HH approximation breaks down; need stiff solvers |
| Phenomenological | Mitchell-Schaeffer, Aliev-Panfilov | minimal AP shape; fast for fibrillation studies |

### Bidomain (rigorous version)

For defibrillation/electrode studies:
```
   ∇·(σi ∇φi)        = β · Cm · ∂Vm/∂t + β · I_ion
   ∇·((σi+σe) ∇φe)   = −∇·(σi ∇Vm)
   Vm                = φi − φe
```
~3× more expensive than monodomain. Most studies use monodomain unless extracellular potentials matter explicitly.

---

## Layer 2 — Excitation-contraction coupling

### Land 2017 model (modern crossbridge standard)

Calcium binding to troponin C:
```
   dTRPN          ⎡ ⎛  [Ca]i  ⎞^n                       ⎤
   ───── = k · ⎢ ⎜───────────⎟    · (1 − TRPN) − TRPN ⎥
    dt          ⎣ ⎝ Ca50(λ)   ⎠                         ⎦
```

Crossbridge cycling (XS = strongly-bound, XW = weakly-bound):
```
   dXS/dt = k_ws · XW   − (k_su + γ_wu) · XS
   dXW/dt = k_uw · (1 − TRPN^(n/2)) − (k_ws + k_wu) · XW
```

Active tension (algebraic):
```
   Ta = T_ref · h(λ) · TRPN^β · (XS + γ_w · XW)
```

| symbol | meaning |
|---|---|
| TRPN | fraction of Ca-bound troponin |
| XS, XW | crossbridge populations |
| λ | sarcomere stretch (= 1 at rest) |
| h(λ) | thin/thick-filament overlap; zero outside ~[0.85, 1.15] |
| T_ref | reference peak tension, ~120 kPa |

### Niederer tanh model (phenomenological alternative)

Closed-form, no calcium dependence:
```
   Ta = T_peak · φ(λ) · tanh²(t_s/τ_c) · tanh²((t_dur − t_s)/τ_r)
        for 0 < t_s < t_dur

   φ(λ) = tanh(ld · (λ − λ₀))     ← length-dependence
   t_s  = t − t_act − t_emd        ← time since activation
```

Cheaper, easier to fit clinical pressure data, but blind to drug effects on Ca handling.

### Other families

| Model | Cooperativity | Use case |
|---|---|---|
| Land 2012 (rat/rabbit) | partial | small mammals |
| Land 2017 (human) | partial | human ventricle (default modern choice) |
| Rice et al. | full crossbridge cycle | high-fidelity force-frequency relationships |
| Lumens | very simple | clinical models, fast |
| Niederer tanh | none (activation-only) | clinical pressure fitting |

---

## Layer 3 — Mechanical deformation

### Cauchy momentum balance

Quasi-static (heart contraction is slow vs elastic waves):
```
   ∇_X · P + b = 0
```

| symbol | meaning |
|---|---|
| P | first Piola-Kirchhoff stress tensor |
| b | body force (usually 0) |
| ∇_X | gradient in reference (undeformed) frame |
| F = I + ∇_X u | deformation gradient |
| u | displacement field |

Full dynamic version with inertia (rarely needed):
```
   ρ₀ · ∂²u/∂t² = ∇_X · P + b
```

### Stress decomposition

```
   P = P_passive(F) + Ta · F · (f₀ ⊗ f₀)
                     ↑
                    rank-1 along the local fiber direction
```

- Active part is rank-1: tension acts only along fiber direction f₀.
- Ta comes from Layer 2.
- Passive part comes from a hyperelastic strain-energy function.

### Passive constitutive laws

```
   P_passive = ∂Ψ/∂F
```

Two dominant choices for myocardium:

#### Holzapfel-Ogden (2009) — orthotropic, current standard

```
              a                          a_f
   Ψ  =  ──── (e^[b(I₁−3)] − 1)  +  ──── (e^[b_f·(I_4f−1)²] − 1)
          2b                          2b_f

         a_s                          a_fs
      + ──── (e^[b_s·(I_4s−1)²] − 1) + ──── (e^[b_fs·I_8fs²] − 1)
         2b_s                          2b_fs

   I₁    = tr(C),   C = FᵀF      (right Cauchy-Green tensor)
   I_4f  = f₀ · C · f₀            (fiber stretch²)
   I_4s  = s₀ · C · s₀            (sheet stretch²)
   I_8fs = f₀ · C · s₀            (fiber-sheet shear)
```

8 parameters; captures fiber, sheet, and fiber-sheet shear stiffness independently.

#### Guccione (1995) — transversely isotropic, simpler

```
   Ψ = (C/2) · (e^Q − 1)

   Q = b_f · E_ff²
     + b_t · (E_ss² + E_nn² + 2·E_sn²)
     + b_fs · 2 · (E_fs² + E_fn²)

   E = ½(C − I)            ← Green-Lagrange strain tensor
```

4 parameters; widely used in earlier studies.

### Incompressibility

Myocardium is ~95% water → near-incompressible:
```
   J = det(F) = 1
```

Two common enforcement strategies:

```
   1. Lagrange multiplier (mixed formulation)
      P = ∂Ψ/∂F − p · F⁻ᵀ        ← p is the hydrostatic pressure
      requires LBB-stable element pair (P1-P0, MINI, P2-P1)

   2. Penalty (compressible model)
      add −κ · (J − 1)² to Ψ
      simpler but ill-conditioned for κ → ∞
```

### Boundary conditions

```
   u = u_base                       on Γ_base       (fix base)
   P · N = −p_LV(t) · J · F⁻ᵀ · N   on Γ_endo,LV    (cavity pressure load)
   P · N = −p_RV(t) · J · F⁻ᵀ · N   on Γ_endo,RV
   P · N = 0  (or pericardium)      on Γ_epi        (free surface or contact)
```

Cavity pressures p_LV(t), p_RV(t) come from a 0D circulation model coupling preload (venous return) and afterload (vasculature) — typically a 3-element windkessel per ventricle.

---

## Coupling: mechanics → electrics

When the tissue deforms, the diffusion operator transforms via the deformation gradient:

```
   D'(F) = J⁻¹ · F · D · Fᵀ
```

So the EP equation in deformed coordinates uses D'(F) instead of D. This is the only obligatory backward arrow from Layer 3 to Layer 1.

### Optional mechano-electric feedback (MEF)

```
   I_ion ← I_ion + I_SAC(λ, Vm)        ← stretch-activated channel current
```

Adds a stretch-modulated current to the ionic model. Important for:
- Commotio cordis (mechanical impact triggering arrhythmia)
- Heart-failure dilated chambers
- Stretch-induced premature ventricular contractions

---

## Full coupled system

```
┌────────────────────────────────────────────────────────────────┐
│ β·Cm · ∂Vm/∂t = ∇·(D'(F) ∇Vm) − β·I_ion(Vm, s) + I_stim       │  parabolic PDE
│ ds/dt = f(Vm, s)                                               │  ODE
│ d(state)/dt = g([Ca]i, λ, state)                               │  ODE
│ Ta = h(state, λ)                                               │  algebraic
│ ∇_X · [P_passive(F) + Ta·F·(f₀⊗f₀) − p F⁻ᵀ] = 0                │  elliptic PDE
│ det(F) = 1                                                     │  constraint
└────────────────────────────────────────────────────────────────┘

   Time scales:                Length scales:
   • EP upstroke    ~1 ms      • cell        ~100 µm
   • full AP        ~300 ms    • element     ~200 µm – 1 mm
   • Ca transient   ~500 ms    • organ       ~10 cm
   • contraction    ~500 ms
   • full beat      ~800 ms
```

### Operator splitting within layers

EP-only (already standard):
```
   for each EP step (Δt ~ 0.01 ms):
     diffuse:  Vm* = Vm + Δt · ∇·(D ∇Vm)        [implicit, Crank-Nicolson]
     react:    Vm  = Vm* + Δt · (−I_ion / Cm)   [explicit, sub-stepped]
              s   = ODE_step(Vm, s)             [Rush-Larsen for gates,
                                                 markov_be / cvode for stiff parts]
```

### Staggered EM coupling (most common)

```
   for each mechanics step (Δt_mech ~ 1 ms):
     advance EP for Δt_mech with previous-step geometry
     compute Ta from updated [Ca]i and λ
     solve mechanics → new u, F (Newton-Raphson, ~5-10 iterations)
     update geometry, recompute D'(F)
```

Variants:
- **Loosely coupled**: one mechanics solve per outer step. Cheap, may need small Δt_mech.
- **Strongly coupled**: iterate EP+mechanics until converged within each outer step. More robust, more expensive.
- **Monolithic**: one global Newton solve. Largest linear systems, rarely used.

### Time-scale separation

EP needs Δt ~ 0.01 ms (sub-millisecond AP upstroke).
Mechanics can use Δt ~ 1 ms (slower contraction kinetics).
Ratio of ~100× is exploited: many EP sub-steps per mechanics step.

### Cost breakdown (typical organ-scale study)

```
   EP (Layer 1)     ~10-30%    of wall time
   Mechanics (3)    ~60-80%    (nonlinear hyperelasticity, Newton iterations)
   Layer 2          ~ <5%      (ODE per node)
   Mesh deformation update ~5-10%
```

Mechanics is almost always the bottleneck. P1-P0 mixed elements with algebraic multigrid for the linearized blocks is the typical solver choice.

---

## Computational tools

### Lineage of the CARP family

```
1990s-2000s            2010+                  2018+              2019-2020+
─────────────────  ─────────────────  ─────────────────  ─────────────────
   CARP        →      CARPentry      →    NumeriCor GmbH         openCARP
 (proprietary,       (proprietary,        (company)              (open-source,
  Vigmond/Plank,      adds mechanics       sells                  EP only,
  EP only)            + hemodynamics       CARPentry-Pro          backward-compat
                      to CARP)             — current product)     with CARP/CARPentry)
```

- **CARP**: original cardiac arrhythmia research package, ~50 person-years of development.
- **CARPentry** (since 2010): extends CARP with multiscale mechanics + hemodynamics.
- **NumeriCor GmbH** (founded 2018): Austrian company, founders Plank, Neic, Augustin, Vigmond. Spun out of Medical University of Graz + University of Bordeaux.
- **CARPentry-Pro**: NumeriCor's commercial product. >100 users, >120 publications.
- **openCARP** (2020): open-source, EP-only, input-compatible with CARP/CARPentry. The version installed in `/usr/local/bin/openCARP`.

### Open-source tools

| Tool | Layer 1 | Layer 2 | Layer 3 | Hemodyn | Notes |
|---|---|---|---|---|---|
| **openCARP** | ✅ | ✅ (plugins) | ❌ | ❌ | C++ + Python (carputils). What's installed locally. |
| **Chaste / Cardiac Chaste** | ✅ | ✅ | ✅ | partial | Oxford C++ + PETSc/Trilinos. Single integrated codebase. |
| **FEniCS / FEniCSx** | via cbcbeat | – | via `pulse` | – | Generic FEM; build your own coupling. Most flexible. |
| **LifeV** | ✅ | ✅ | ✅ | ✅ | EPFL/Polimi C++. Strong on FSI. |
| **Cardioid** | ✅ | partial | – | – | LLNL exascale-capable. |

### Commercial tools

| Tool | Layer 1 | Layer 2 | Layer 3 | Hemodyn | Notes |
|---|---|---|---|---|---|
| **CARPentry-Pro** (NumeriCor) | ✅ | ✅ | ✅ | ✅ | Input-compatible with openCARP. The reference modern commercial cardiac-EM tool. |
| **Continuity** (UCSD) | ✅ | ✅ | ✅ | partial | Mature platform, web portal + binaries. |
| **SIMULIA Living Heart** (Dassault) | ✅ | ✅ | ✅ | ✅ | Prepackaged whole-heart EM in Abaqus. Industry/regulatory. |
| **Abaqus / Ansys + UMAT** | – | via UMAT | ✅ | – | Generic FE codes with user subroutines for active stress. |

### What's available locally

```
/usr/local/bin/openCARP        ← Layer 1 monodomain/bidomain solver
/usr/local/bin/bench           ← Layer 1+2 single-cell solver
                                  (Stress_Land17, Stress_Niederer, ... plug-ins)
/usr/local/bin/meshtool        ← preprocessing
/usr/local/meshalyzer          ← visualization
```

### Paths to add Layer 3 (mechanics)

If you want to do full EM with what you have:

1. **Install FEniCS + `pulse`**:
   ```
   pip install fenics-pulse
   ```
   Write the staggered coupling glue yourself: openCARP outputs Ta and Cai, FEniCS solves the elasticity, feeds back λ and F.

2. **Apply for a CARPentry-Pro license** at NumeriCor — same input file format as openCARP, mechanics included. Fastest path to a published-quality EM simulation.

3. **Use Chaste** — different input format, but full EM in one tool. Steeper learning curve.

4. **Prescribe deformation** — for many studies you don't need to solve Layer 3; hand bench/openCARP a λ(t) trace from imaging or a separate mechanics simulation. That's exactly what `bench --fs lambda.dat` does in the 06_EM_coupling tutorial. Cheap; sufficient if your scientific question is how mechanics affects electrics, not the converse.

---

## References

- openCARP project: <https://opencarp.org/>
- openCARP paper: Plank et al., *Comput Methods Programs Biomed* 208 (2021) 106223. <https://www.sciencedirect.com/science/article/abs/pii/S0169260721002972>
- NumeriCor GmbH: <https://www.numericor.at/>
- CARP → openCARP transition guide: <https://opencarp.org/documentation/carp-carpentry-user>
- Holzapfel-Ogden myocardium model: Holzapfel & Ogden, *Phil Trans R Soc A* 367 (2009) 3445.
- Land 2017 cardiac mechanics model: Land et al., *J Mol Cell Cardiol* 106 (2017) 68.
- Niederer tanh stress: Niederer et al., *Cardiovasc Res* 89 (2011) 336.
- Guccione passive law: Guccione et al., *J Biomech Eng* 117 (1995) 152.
