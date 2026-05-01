# INDEX — agent navigation map

Compact sitemap. An agent reads this once to plan retrieval, then opens the relevant note(s).

## Navigation by question

| Question shape | Go to |
| --- | --- |
| "What is X?" / "How does X work mathematically?" | `concepts/` |
| "Which tool for Y?" / "How do I use tool Z?" | `tools/` |
| "How do I run a Y experiment?" | `workflows/` |
| "What did we find when we ran Z?" / "What's a typical value of W?" | `findings/` |
| "X is broken / errors / behaves weirdly" | `troubleshooting/` |
| "Where's the source paper for X?" | `refs/` |
| Need to actually run a tutorial | `opencarp_tutorials/` (write-through to install) |

## Top-level files (current)

| Path | Status | Use when |
| --- | --- | --- |
| `concepts/em-coupling.md` | mixed | Question about Vm → Ca → tension → deformation; choosing an EM tool |
| `troubleshooting/meshalyzer-haswell.md` | validated | Meshalyzer shader errors or broken vertex picking on Intel Haswell / WSL |

## Environment shortcuts

```
openCARP            /usr/local/bin/openCARP
bench               /usr/local/bin/bench
meshtool            /usr/local/bin/meshtool
meshalyzer          /usr/local/meshalyzer/meshalyzer  (patched, see troubleshooting/)
carputils python    /home/cdc/miniconda3/envs/opencarp/bin/python

PATH-required tools (need opencarp env on PATH):
  tuneCV, restituteCV, cellml_converter.py, limpet_fe.py, make_dynamic_model.sh
```

## Tutorial knowledge (live in install, accessible via opencarp_tutorials/)

Per-tutorial `CLAUDE.md` files in each tutorial directory hold detailed setup + run results + lessons. Top-level summaries:

- `opencarp_tutorials/01_EP_single_cell/PROGRESS.md` — 9 tutorials swept
- `opencarp_tutorials/02_EP_tissue/PROGRESS.md` — 23 tutorials swept

When the agent needs an end-to-end summary (cross-tutorial), see `findings/` (to be populated).

## Retrieval tips for agents

- Start at this INDEX, not at `README.md` — the README is for humans.
- A directory's `README.md` is its index — open before diving into individual notes.
- Frontmatter `use_when:` field is the primary retrieval signal — match the user's question against it.
- `status: validated` notes can be cited directly. `status: borrowed-from-paper` should be cited with the paper as the authority, not this library.
- If the answer would require updating a note, do it. If a note is missing, add it under the right directory rather than embedding the answer only in the conversation.
