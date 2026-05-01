# CLAUDE.md — 01_EP_single_cell / 04_limpet_fe

## What this tutorial demonstrates

End-to-end loop for adding a custom ionic model:

1. Author the model in **EasyML** (`.model` file).
2. Translate to C++ with `limpet_fe.py`.
3. Compile to a runtime-loadable shared library with `make_dynamic_model.sh`.
4. Run with `bench --load-module <path>.so` and exercise parameters via `--imp-par`.

The shipped example is the **modified Beeler-Reuter (Drouhard-Roberge)** model in `my_MBRDR.model`, with one parameter `APDshorten` exposed via `.param()` markup.

## Setup

| | value | source |
| --- | --- | --- |
| Source `.model` | `my_MBRDR.model` (DR variant of Beeler-Reuter, ICaL/d/f rates scaled by `APDshorten`) | shipped |
| Translator | `/usr/local/bin/limpet_fe.py` | on PATH |
| Compiler wrapper | `/usr/local/bin/make_dynamic_model.sh` | on PATH |
| Default model param | `APDshorten=3` (in run.py, NOT in the .model where default is 1) | run.py:285-288 |
| Default BCL | 1000 ms | `--bcl` |
| numstim | 1 | `--numstim` |
| dt | 0.025 ms | `--dt` |

## Available locally

- `make_dynamic_model.sh`: yes — `/usr/local/bin/make_dynamic_model.sh` (alias of `/usr/local/lib/opencarp/share/openCARP/physics/limpet/src/make_dynamic_model.sh`).
- `limpet_fe.py`: yes — `/usr/local/bin/limpet_fe.py`.
- Required openCARP source headers: present at `/usr/local/lib/opencarp/share/openCARP/physics/limpet/src/`. The wrapper uses these include paths automatically.
- Build emits a warning `Cannot determine build info, not using CVODE. Use _build dir when using CMake to make it work.` — this is harmless for the simple example (no CVODE-required integration).

## Findings: Run results (2026-04-28)

### Step 1+2 — Translate + compile

```
cd /usr/local/lib/opencarp/share/tutorials/01_EP_single_cell/04_limpet_fe
make_dynamic_model.sh my_MBRDR
```

Produces:

- `my_MBRDR.cc`, `my_MBRDR.h` — generated C++ source
- `my_MBRDR_dyn.cc` — dynamic-loader wrapper
- `my_MBRDR.so` — compiled shared object (~150 KB)

The full gcc invocation used (extracted from the wrapper output):

```
/usr/bin/gcc -fPIC -shared -std=c++14 -lm \
  -I/usr/local/lib/opencarp/share/openCARP/physics/limpet/src \
  -I/usr/local/lib/opencarp/share/openCARP/_build/physics/limpet \
  -I/usr/local/lib/opencarp/share/openCARP/_build/simulator \
  -I/usr/local/lib/opencarp/share/openCARP/numerics \
  -I/usr/local/lib/opencarp/share/openCARP/simulator \
  -I/usr/local/lib/opencarp/share/openCARP/fem \
  -I/usr/local/lib/opencarp/share/openCARP/fem/slimfem/src \
  -I/usr/local/lib/opencarp/share/openCARP/param/include \
  -I/usr/local/lib/opencarp/share/openCARP/../../lib/petsc/include \
  -I/usr/local/lib/opencarp/share/openCARP/../../lib/openmpi/include \
  -DMY_MBRDR_CPU_GENERATED \
  -o my_MBRDR.so my_MBRDR.cc my_MBRDR_dyn.cc
```

### Step 3 — Run with bench

```
./run.py --load-module ./my_MBRDR.so --imp-par 'APDshorten=1' --ID 2026-04-28_apd1
./run.py --load-module ./my_MBRDR.so --imp-par 'APDshorten=3' --ID 2026-04-28_apd3
```

Both runs: 1 stim @ t=1 ms, 1000 ms duration, dt=0.025 ms. Wall time ~0.05 s each.

| Run | APD90 (ms) | Cai peak (µM) | Vm peak (mV) | Vm rest (mV) |
| --- | --- | --- | --- | --- |
| `APDshorten=1` | **234.0** | 6.09 | 38.5 | −86.9 |
| `APDshorten=3` | **128.0** | 5.48 | 38.5 | −86.9 |

`APDshorten` scales the α/β rates of the `d` and `f` gates of the slow inward (Ca) channel — tripling them accelerates ICa kinetics, sharply shortening the plateau (234 → 128 ms = 45% reduction). Cai peak is only mildly affected.

## Bug: run.py default `--imp-par` masks "no modifier" baseline

`04_limpet_fe/run.py:285-288`:

```python
group.add_argument('--imp-par',
                   type = str,
                   default = 'APDshorten=3',
                   help = 'ionic model adjustments')
```

Calling `./run.py --load-module ./my_MBRDR.so` (no explicit `--imp-par`) silently runs with `APDshorten=3`, **not** the model's intrinsic default of 1. To get the baseline behaviour (the `.param() { APDshorten = 1 }` default in the .model file), you must explicitly pass `--imp-par 'APDshorten=1'`. Both my initial runs above (default + APDshorten=3) gave identical APD90=128 ms because of this.

## Lessons (transferable)

1. **`.so` lives next to the .model**: `--load-module ./my_MBRDR.so` (with `./` prefix) — the loader requires an absolute-style path, not a name-only lookup, when not in a system search path.
2. **`.param()` markup in the .model** maps directly to `--imp-par <name>=<value>` overrides at runtime. Defaults declared in `.param() { … = N; }` apply unless overridden. Inspect available knobs of a compiled model with `bench --imp-info <name>` after loading the .so — but that requires a different invocation pattern (load via the standard mechanism, not as an external module). Easier: read the `.param()` block.
3. **Output-file naming for external models**: when a custom model named `my_MBRDR` is loaded, bench writes `<EP>.t.bin`, `<EP>.Vm.bin`, plus per-state-variable `<EP>_<EP>.<sv>.bin` files (where `<EP>=my_MBRDR`). State variables come from the `.trace()` group + the public state vars listed in the model.
4. **The harmless CVODE warning** at compile time only matters if your model uses `.method(cvode);` — most simple Hodgkin-Huxley-style models like this one use forward Euler / Rush-Larsen and don't need CVODE.
5. **Generated `.cc` and `.h`** stay in the tutorial directory after a successful build; they can be inspected/edited if you want to manually tune integration methods or add code that EasyML can't express.
