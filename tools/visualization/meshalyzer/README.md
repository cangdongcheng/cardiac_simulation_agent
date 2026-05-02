# tools/visualization/meshalyzer/

Meshalyzer is openCARP's built-in 3D viewer for meshes and time-series field data (`.igb`).

**Install:** `/usr/local/meshalyzer/meshalyzer` (extracted from the v6.1 AppImage)
**Add to PATH:** `export PATH=/usr/local/meshalyzer:$PATH`

## Current notes

| File | Status | Topic |
| --- | --- | --- |
| `vertex-picking-fixes.md` | validated | Shader compilation + vertex picking fixes for Intel Haswell, WSL, and Windows Cygwin |

## Likely additions

- `usage.md` (or expand this README) — basic invocation: `meshalyzer <mesh> <data.igb> [view.mshz]`. Mouse controls, vertex picking, animation.
- `mshz-views.md` — saved view files. How to author one, common patterns (camera position, color map, clip planes).
- `troubleshooting.md` — for issues other than the shader / picking bugs.

When the user runs into shader compilation errors or broken vertex picking, open `vertex-picking-fixes.md` directly — that note has the symptoms in its `use_when:` frontmatter.
