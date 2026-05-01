# findings/

The **LEARNED**: validated results from experiments. Distilled lessons, not raw outputs.

A note belongs here if it's the kind of statement that would otherwise be re-derived every time someone asks ("what's a typical CV in a TTP slab?", "how steep is the TTP S1S2 slope?", "how long must I pace before reaching limit cycle?").

These notes ARE evidence. Always cite the exact run (path, date, command) so the claim is reproducible.

## Structure for a findings note

```
- Headline finding (one sentence)
- How it was measured (mesh, model, command, date)
- Numerical result (with units)
- Cross-references: which concepts explain it, which workflow produced it
```

## Current notes

(none yet — to be populated by distilling per-tutorial CLAUDE.md content)

## Likely additions (high priority — content already exists in tutorial CLAUDE.mds)

- `tissue-tutorial-sweep.md` — distilled summary of all 23 02_EP_tissue findings: CV numbers, APD baselines, λ values, reentry protocols, periodic-BC behavior. Source: `opencarp_tutorials/02_EP_tissue/PROGRESS.md` and per-tutorial `CLAUDE.md`.
- `single-cell-sweep.md` — distilled summary of all 9 01_EP_single_cell findings: TTP baseline APD90, peri-infarct alternans, restitution slopes per model, EM coupling tension peaks. Source: `opencarp_tutorials/01_EP_single_cell/PROGRESS.md` and per-tutorial `CLAUDE.md`.
- `benchmarks.md` — runtime numbers (real-time factors per ionic model), memory usage per mesh resolution, bidomain vs monodomain cost.

## Likely additions (lower priority)

- `ionic-model-comparison.md` — APD, peak Vm, rest Vm side-by-side across TTP, Courtemanche, GPB, ToR-ORd, etc.
- `fiber-orientation-effects.md` — sensitivity of CV/APD to fiber angle assumptions.

## Distinction from `opencarp_tutorials/.../CLAUDE.md`

The per-tutorial `CLAUDE.md` files are the **raw record** of running each tutorial. They live in the install tree (write-through) for proximity to the source.

The notes here are the **distilled cross-tutorial knowledge** — the kind of summary you'd want as a one-page handout. We don't duplicate; we synthesize.
