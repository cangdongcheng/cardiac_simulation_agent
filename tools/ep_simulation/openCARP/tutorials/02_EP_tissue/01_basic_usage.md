# CLAUDE.md — 02_EP_tissue / 01_basic_usage

## What this tutorial demonstrates

The most basic openCARP tissue experiment: a single transmembrane stimulus on one face of a thin slab, contrasting subthreshold (depolarizing and hyperpolarizing) responses with a suprathreshold response that triggers a propagating action potential. This is the canonical "did my install work, can I get a wave to propagate" sanity test.

## Setup

| | value | source |
| --- | --- | --- |
| Mesh | `mesh.Block(size=(10, 1, 0.2))` mm | `run.py:192` |
| Fibres | longitudinal (0°/0°/90°/90°) | `run.py:195` |
| Solver | monodomain (`bidomain=0`) | `basic.par` |
| Ionic model | DrouhardRoberge (no parameter modification) | `basic.par` |
| Pulse start | t=0 (default from `basic.par:13`) | `basic.par` |
| Sim integration | `dt=25` µs (override in `run.py:228`) | `run.py:228` |
| Stim electrode | box at x∈[−5050,−4950], y∈[−550,550], z∈[−150,150] µm — covers full y-z at the −x face | `run.py:215-220` |
| `--duration` | sim length (ms), default 20 | parser |
| `--S1-strength` | µA/cm² ≡ pA/pF, default 20 | parser |
| `--S1-dur` | ms, default 2 | parser |

`crct.type=0` here is **transmembrane current**, not "unset" — context-dependent meaning. In bidomain stim configs (tutorial 02) it acts as a placeholder; here it's the active stimulus mode.

Mesh resolution: 3333 nodes for the 10×1×0.2 mm slab (≈170 µm spacing).

## Sub-experiments and CLI

| exp | command | expected response |
| --- | --- | --- |
| exp01 subthreshold + | `./run.py --duration 20 --S1-strength 20 --S1-dur 15` | passive depolarization, no AP |
| exp02 subthreshold − | `./run.py --duration 20 --S1-strength -20 --S1-dur 15` | passive hyperpolarization, mirror of exp01 |
| exp03 suprathreshold | `./run.py --duration 20 --S1-strength 250 --S1-dur 2` | AP triggered, propagates across slab |

`--ID exp01` etc. is useful to override the default jobID (which is just date+duration and would collide across runs in the same day).

## Findings (run on 2026-04-28)

Resting Vm = −86.93 mV. Vm sampled at three node bands: **stim** (x<−4500, 67 nodes), **mid** (|x|<200), **far** (x>4500). All three exps complete in <0.3 s electrics compute.

| exp | global Vm range | stim site peak/trough | mid Vm | far Vm | AP triggered? |
| --- | --- | --- | --- | --- | --- |
| exp01 (+20, 15 ms) | −86.93 / −78.78 | peak −80.47 @ 14 ms (~6 mV ↑) | flat ~−86 | flat ~−86 | **no** — passive only |
| exp02 (−20, 15 ms) | −92.69 / −86.09 | trough −91.07 @ 14 ms (~4 mV ↓) | flat ~−86 | flat ~−86 | **no** — passive mirror |
| exp03 (+250, 2 ms) | −86.93 / +44.44 | peak +35.58 @ 3 ms | peak +36.50 @ 11 ms | peak +38.67 @ 18 ms | **yes**, propagating |

Notes:
- Subthreshold responses are local (no spatial spread visible at this binning); response decays back toward rest after stim ends. The slight asymmetry between exp01 (+6 mV from rest) and exp02 (−4 mV from rest) reflects the rectifying behavior of cardiac K⁺ currents — the membrane is somewhat "stiffer" against hyperpolarization, even subthreshold.
- exp03 shows a clean planar wave: peak Vm reached at the stim site at t=3 ms (during/just after the 2 ms pulse), then sweeps across to the far end by t=18 ms. Crude CV estimate from peak-Vm timing: 1 cm / 15 ms ≈ **0.67 m/s** (rough — proper CV uses LAT at −10 mV crossing).
- Final-frame Vm increases monotonically with x (stim +10.84 < mid +15.63 < far +33.19), consistent with the wave still being in late depolarization / plateau at the far end while the stim site has already started repolarizing.

## Lessons (transferable)

1. **`crct.type=0` is overloaded.** It means transmembrane current when used as an actual stimulus, but acts as an inert placeholder when an extracellular `crct.type` is overridden later (cf. tutorial 02). Always interpret in context.
2. **Default `dt=25` µs (override of `basic.par`).** This is the openCARP integration timestep; not the same as `spacedt`/`timedt` (output stride). Reducing `dt` is the first knob to try if you suspect numerical instability in the upstroke.
3. **Mesh `_, etags, _ = txt.read(meshname + '.elem')` then filtering tag 0** is the idiomatic way to derive `IntraTags`/`ExtraTags` from a generated block. Tag 0 is conventionally bath/extracellular-only; non-zero tags are myocardium.
4. **Subthreshold strength-duration trade-off**: 20 µA/cm² × 15 ms (exp01, charge = 0.3 mC/cm²) fails to capture, but 250 µA/cm² × 2 ms (exp03, charge = 0.5 mC/cm²) succeeds. This isn't just about total charge — peak current also matters (the rheobase/chronaxie story).
