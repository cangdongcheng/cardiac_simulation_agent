# cardiac_simulation_agent

A validated knowledge library for cardiac simulation, structured for an agent to pick up the entire pipeline quickly.

- **Rules / contributor guide** → `CLAUDE.md`
- **Agent navigation map** → `INDEX.md`
- **Working tutorial tree** → `opencarp_tutorials/` (write-through symlink to the openCARP install)

## Layout

```
cardiac_simulation_agent/
├── CLAUDE.md                ← rules (constitution)
├── README.md                ← you are here
├── INDEX.md                 ← agent navigation map
│
├── workflows/               ← goal-oriented: tool-chains with stated endpoints
├── tools/                   ← tool-oriented: per-tool usage, by category
│   ├── ep_simulation/       (openCARP, bench, carputils)
│   ├── mesh_processing/     (meshtool, cobiveco, ...)
│   ├── visualization/       (meshalyzer, ...)
│   └── em_coupling/         (placeholder)
│
├── opencarp_tutorials → /usr/local/lib/opencarp/share/tutorials
│                            ← write-through symlink to the install
└── refs/                    ← shared bibliography
```

## Two questions, two doors

| Question | Open |
| --- | --- |
| "How do I go from X to Y?" (a goal) | `workflows/` |
| "How does tool Z work?" (a tool) | `tools/<category>/` |

Concepts, findings, and bugs aren't separate categories — they're sections inside whichever workflow or tool note they belong to.

## Local environment

| | |
| --- | --- |
| openCARP, bench, meshtool | `/usr/local/bin/` |
| meshalyzer | `/usr/local/meshalyzer/` (patched, see `tools/visualization/meshalyzer/haswell-fix.md`) |
| carputils Python | `/home/cdc/miniconda3/envs/opencarp/bin/python` |
| Tutorial source tree | `opencarp_tutorials/` (write-through to install) |

The `opencarp` conda env must be on `PATH` for `tuneCV`, `restituteCV`, `cellml_converter.py`, `limpet_fe.py`, `make_dynamic_model.sh` to resolve.
