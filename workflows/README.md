# workflows/

**Goal-oriented** notes: tool-chains with explicit endpoints. Each note answers "how do I go from X to Y?"

A workflow note must state at the top:
- **Input** — what you start with
- **Output** — what you end up with
- **Tools** — the chain, in order

See `../CLAUDE.md` § "Workflow note structure" for the required layout.

## Current notes

| File | Status | Pipeline |
| --- | --- | --- |
| `em-coupling.md` | mixed | Vm trace → Ca dynamics → active tension → tissue deformation. Math + tool landscape. |

## Likely additions (write when a workflow is actually exercised end-to-end)

- `mesh-to-simulation.md` — raw geometry → fiber-tagged mesh → AP propagation traces
- `single-cell-pacing.md` — ionic model + BCL → limit-cycle `.sv` → reusable initial state
- `apd-restitution.md` — initial state → S1S2 / Dynamic protocol → APD-vs-DI curve + slope
- `tissue-conduction-tuning.md` — slab mesh + target CV → tuned conductivities (gi, ge)
- `parameter-sweep.md` — parameter ranges → polling file → population of model outputs
- `reentry-induction.md` — baseline tissue → S1-S2-S3 protocol → induced reentry
- `lhs-pom.md` — parameter ranges → Latin Hypercube samples → distribution of model outputs
- `cellml-import.md` — `.cellml` model → EasyML → compiled `.so` → bench run

Each new workflow should be derived from a tutorial we've actually run, with the relevant `tools/ep_simulation/openCARP/tutorials/<section>/<tutorial>.md` linked as the raw record.
