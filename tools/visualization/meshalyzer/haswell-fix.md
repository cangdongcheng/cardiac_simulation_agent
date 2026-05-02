---
title: Meshalyzer vertex picking fixes for Intel Haswell / older Mesa / WSL / Windows (Cygwin)
status: validated   # Haswell binary patch + WSL source fix + Windows Cygwin binary patch all confirmed working
use_when: meshalyzer fails to compile shaders, prints `ERROR::SHADER::VERTEX::COMPILATION_FAILED`, or vertex picking does not select nodes — Intel Haswell, WSL, or Windows
last_updated: 2026-05-02
---

# Meshalyzer vertex picking fixes for Intel Haswell / older Mesa / WSL / Windows (Cygwin)

Documents the patches applied on `cdc-ThinkPad-X240` (Ubuntu 24.04, Intel HD Graphics 4400), on WSL2 (llvmpipe software renderer), and on the Windows Cygwin build to fix vertex picking.

## Symptoms

When running `/usr/local/meshalyzer/meshalyzer <mesh> <data.igb>`:

1. Terminal prints repeated shader errors:
   ```
   ERROR::SHADER::VERTEX::COMPILATION_FAILED
   0:2(12): warning: extension `GL_EXT_clip_cull_distance' unsupported in vertex shader
   0:5(7): error: redeclaration cannot change qualification of `gl_ClipDistance'
   ERROR::SHADER::PROGRAM::LINKING_FAILED
   error: linking with uncompiled/unspecialized shader
   ```
   (Other variants: `0:1(10): error: GLSL 4.10 is not supported`, `0:39(2): error: unsized array index must be constant`.)

2. Mesh and vm.igb render fine, animation plays — but **clicking on the mesh never selects a vertex**. Hovering only highlights surface fill.

## Root cause

Two independent bugs:

| # | Bug | Where |
|---|---|---|
| A | Shaders declare `#version 410 core` but the driver only supports up to GLSL 4.00 (Intel Haswell maxes at OpenGL 4.0). All version-410 shaders fail to compile. | All visual + picking shaders |
| B | The **picking shader** (separate render pass that encodes vertex IDs as colors so the click handler can decode "which vertex did the user click") declares `float gl_ClipDistance[6];` *without* the `out` qualifier. The built-in `gl_ClipDistance` is implicitly `out`, so Mesa correctly rejects this as `redeclaration cannot change qualification`. This is a meshalyzer source bug, not a driver issue. | `src/select_util.h::get_select_shader()` in meshalyzer 6.1 |

Bug A makes everything fail spectacularly. Bug B is the reason picking *specifically* still doesn't work even after Bug A is fixed: rendering uses a different shader pipeline than picking, so visual rendering can succeed while picking silently fails.

## Fix overview

The shaders are GLSL string literals baked into the meshalyzer binary. Patch the binary in-place, preserving byte length so other offsets stay valid.

| # | Original string (bytes) | Replacement | Count | Reason |
|---|---|---|---|---|
| 1 | `#version 410 core` (17) | `#version 400 core` | 11 | Driver only supports GLSL ≤4.00 |
| 2 | `#extension GL_EXT_clip_cull_distance : enable\n` (46) | `<spaces>\n` | 1 | Extension unsupported on Mesa Intel |
| 3 | `float gl_ClipDistance[6];\n` (26) | `<spaces>\n` | 1 | Removes the no-`out` redeclaration that triggers the qualifier error |
| 4 | `  for( int i=0; i<6; i++ )\n    gl_ClipDistance[i] = dot(vec4(aPos,1.0), clipEqn[i]);\n` (84) | `<spaces>` | 1 | The for-loop body would error with "unsized array index must be constant" once #3 is removed |

Trade-off from patches 3 + 4: the picking shader no longer respects user-defined clip planes. For normal use you'll never notice; only edge case is "click a vertex that's currently clipped away" — picking might select a vertex you can't see.

## Linux binary patch (works on any Linux x86_64 with the v6.1 AppImage)

### 1. Install meshalyzer

```bash
sudo mkdir -p /usr/local/meshalyzer
sudo curl -L "https://git.opencarp.org/api/v4/projects/39/packages/generic/meshalyzer-appimage/6.1/Meshalyzer-6.1-x86_64.AppImage" \
     -o /usr/local/meshalyzer/meshalyzer.appimage
sudo chmod +x /usr/local/meshalyzer/meshalyzer.appimage
```

### 2. Extract the AppImage

The AppImage's binary is read-only inside the squashfs, so we extract it.

```bash
cd /tmp
/usr/local/meshalyzer/meshalyzer.appimage --appimage-extract
sudo cp -r /tmp/squashfs-root /usr/local/meshalyzer/extracted
sudo ln -sf /usr/local/meshalyzer/extracted/AppRun /usr/local/meshalyzer/meshalyzer
```

After this, running `/usr/local/meshalyzer/meshalyzer` invokes the extracted binary.

### 3. Apply the binary patches

```bash
python3 - <<'PY'
target = '/usr/local/meshalyzer/extracted/usr/bin/meshalyzer'
out    = '/tmp/meshalyzer-patched'
data   = bytearray(open(target, 'rb').read())

# Patch 1: downgrade GLSL 4.10 -> 4.00 in all shaders
old, new = b'#version 410 core', b'#version 400 core'
n = 0
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    data[p:p+len(old)] = new
    n += 1
    i = p + 1
print(f'#version 410 -> 400: {n} replacements')

# Patch 2: remove unsupported clip_cull_distance extension
old = b'#extension GL_EXT_clip_cull_distance : enable\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Removed clip_cull_distance extension at offset {p}')

# Patch 3: blank out the no-qualifier gl_ClipDistance redeclaration
# (only the one without preceding 'out ')
old = b'float gl_ClipDistance[6];\n'
new = b' ' * len(old)
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    if data[max(0, p-4):p] != b'out ':
        data[p:p+len(old)] = new
        print(f'Wiped no-out gl_ClipDistance redeclaration at {p}')
    i = p + 1

# Patch 4: blank out the for-loop using gl_ClipDistance[i]
old = b'  for( int i=0; i<6; i++ )\n    gl_ClipDistance[i] = dot(vec4(aPos,1.0), clipEqn[i]);\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Wiped picking for-loop at {p}')

open(out, 'wb').write(data)
print('Wrote', out)
PY
sudo cp /tmp/meshalyzer-patched /usr/local/meshalyzer/extracted/usr/bin/meshalyzer
sudo chmod +x /usr/local/meshalyzer/extracted/usr/bin/meshalyzer
```

Expected output:
```
#version 410 -> 400: 11 replacements
Removed clip_cull_distance extension at offset 2128674
Wiped no-out gl_ClipDistance redeclaration at 2128796
Wiped picking for-loop at 2129102
Wrote /tmp/meshalyzer-patched
```

(Offsets will differ slightly by build, but counts should match: 11 + 1 + 1 + 1.)

### 4. Verify

```bash
cd ~/tutorials/02_EP_tissue/00_simple
/usr/local/meshalyzer/meshalyzer meshes/*/block.pts 2*/vm.igb simple.mshz
```

- No `ERROR::SHADER::*` lines should appear in the terminal.
- Clicking on the mesh should select individual vertices.

## Windows Cygwin binary patch (validated 2026-05-02)

The same binary-patching approach works on the Windows Cygwin build of meshalyzer. The GLSL strings are embedded identically. The FBO fix (Bug C) is **not** needed on native Windows — `glReadPixels(GL_BACK)` works correctly with a real GPU driver.

Binary location: `~/SSD/meshalyzer/meshalyzer.exe`

```bash
cp ~/SSD/meshalyzer/meshalyzer.exe ~/SSD/meshalyzer/meshalyzer.exe.bak

python3 - <<'PY'
target = '/home/cangdongcheng/SSD/meshalyzer/meshalyzer.exe'
out    = '/home/cangdongcheng/SSD/meshalyzer/meshalyzer.exe'
data   = bytearray(open(target, 'rb').read())

# Patch 1: downgrade GLSL 4.10 -> 4.00 in all shaders
old, new = b'#version 410 core', b'#version 400 core'
n = 0
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    data[p:p+len(old)] = new
    n += 1
    i = p + 1
print(f'#version 410 -> 400: {n} replacements')

# Patch 2: remove unsupported clip_cull_distance extension
old = b'#extension GL_EXT_clip_cull_distance : enable\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Removed clip_cull_distance extension at offset {p}')

# Patch 3: blank out the no-qualifier gl_ClipDistance redeclaration
old = b'float gl_ClipDistance[6];\n'
new = b' ' * len(old)
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    if data[max(0, p-4):p] != b'out ':
        data[p:p+len(old)] = new
        print(f'Wiped no-out gl_ClipDistance redeclaration at {p}')
    i = p + 1

# Patch 4: blank out the for-loop using gl_ClipDistance[i]
old = b'  for( int i=0; i<6; i++ )\n    gl_ClipDistance[i] = dot(vec4(aPos,1.0), clipEqn[i]);\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Wiped picking for-loop at {p}')

open(out, 'wb').write(data)
print('Wrote', out)
PY
```

Expected output:
```
#version 410 -> 400: 12 replacements
Removed clip_cull_distance extension at offset 2301354
Wiped no-out gl_ClipDistance redeclaration at 2301476
Wiped picking for-loop at 2301782
```

(12 version replacements vs 11 on Linux — the Cygwin build has one additional shader string. Offsets will differ by build.)

### Reverting (Windows)

```bash
cp ~/SSD/meshalyzer/meshalyzer.exe.bak ~/SSD/meshalyzer/meshalyzer.exe
```

## WSL source-build fix (validated 2026-05-02)

On WSL2 (Ubuntu 24.04, llvmpipe LLVM 20.1.2, Mesa 25.0.7, OpenGL 4.5), the Haswell binary patches alone do **not** fix picking. The GLSL version is not the issue (OpenGL 4.5 supports GLSL 4.50), and the shader compiles without errors. The root cause is different:

### WSL root cause (Bug C)

| # | Bug | Where |
|---|---|---|
| C | The picking pass renders vertex-ID-encoded colors to the **default framebuffer** (back buffer), then `glReadPixels(GL_BACK)` reads it back. On WSL2/WSLg with llvmpipe, `glReadPixels` from the default framebuffer returns stale data (the previously displayed frame, not the just-rendered pick pass). This is a WSLg/Wayland compositor issue — the default framebuffer is not reliably readable after rendering. | `src/select_util.cc::mesh_render()` renders to default FB; `src/Frame.cc::fill_buffer()` reads back from `GL_BACK` |

Additionally, `draw_surfaces()` in `src/TBmeshWin.cc::draw()` is called unconditionally (not guarded by `renderMode&RENDER`), so it draws the mesh surface **over** the picking colors even if the readback worked.

### Diagnosis evidence

Debug output showed:
- `col=(143,143,143,255)` before the `draw_surfaces` guard — reading the gray mesh surface, not pick colors
- `col=(0,0,0,255)` after guarding `draw_surfaces` — reading a black frame with alpha=255 (the pick clear is `(0,0,0,0)` with alpha=0, confirming `glReadPixels` reads a different buffer)

### Source changes applied

All changes are in the source build at `~/meshalyzer/`.

**1. GLSL version downgrade** (same intent as Haswell patches, harmless on 4.5 drivers):

| File | Change |
|---|---|
| `src/render_func.h:4` | `#version 410 core` → `#version 400 core` |
| `src/ColourBar.h` | 4× `#version 410 core` → `#version 400 core` |
| `src/ScreenTxt.h` | 2× `#version 410 core` → `#version 400 core` |

**2. Picking shader cleanup** (`src/select_util.h`):

- Removed `#extension GL_EXT_clip_cull_distance : enable` (unsupported on Mesa Intel/llvmpipe)
- Removed `float gl_ClipDistance[6];` declaration and `clipEqn` uniform + for-loop (trade-off: picking ignores clip planes)

**3. Guard `draw_surfaces` during pick mode** (`src/TBmeshWin.cc:349`):

```diff
-  draw_surfaces( _rd_op, _trans_elems, false );
+  if ( !SELECT(renderMode) )
+    draw_surfaces( _rd_op, _trans_elems, false );
```

**4. FBO-based picking readback** (the actual WSL fix):

`src/select_util.cc::mesh_render()` — instead of rendering to the default framebuffer and relying on `fill_buffer`'s `glReadPixels(GL_BACK)`, the picking pass now:

1. Creates a temporary FBO with RGBA8 color + DEPTH24 renderbuffers sized to the viewport
2. Binds the FBO and renders the pick points into it
3. Calls `glReadPixels` on the FBO (guaranteed to return the just-rendered data)
4. Stores the pixel data in `SelectVtx::_pickbuf`
5. Deletes the FBO

`src/select_util.h` — added `_pickbuf`, `_pick_w`, `_pick_h` members and a `pixel_color(x, y, color)` method to read from the stored buffer.

`src/TBmeshWin.cc::process_hits()` — reads pixel colors from `_select.pixel_color()` instead of `frame.pixel_color()`.

### Rebuild

```bash
cd ~/meshalyzer/_build && make -j$(nproc)
```

The binary at `~/meshalyzer/_build/src/meshalyzer` (symlinked from `/usr/local/meshalyzer/meshalyzer`) is updated in place.

### Why the Haswell binary patch worked without the FBO fix

On Haswell (Intel HD 4400, X11, hardware GL), `glReadPixels(GL_BACK)` from the default framebuffer works correctly. The only issues were the shader compilation errors (Bugs A + B). Once those were patched, picking worked because the readback path was reliable. On WSL/Wayland/llvmpipe, the readback path itself is broken (Bug C), requiring the FBO approach.

## General WSL notes

- AppImages need FUSE in WSL. If the AppImage refuses to mount, use `--appimage-extract` (step 2) and run `/usr/local/meshalyzer/extracted/AppRun` directly. The extracted approach is what we use here anyway, so this isn't extra work.

## Reverting

```bash
sudo rm -rf /usr/local/meshalyzer/extracted /usr/local/meshalyzer/meshalyzer
sudo mv /usr/local/meshalyzer/meshalyzer.appimage /usr/local/meshalyzer/meshalyzer
sudo chmod +x /usr/local/meshalyzer/meshalyzer
```

## Upstream report (recommended)

The picking-shader bug (#B above) is a real meshalyzer issue and should be reported at <https://git.opencarp.org/openCARP/meshalyzer/-/issues>. Suggested patch in `src/select_util.h`:

```diff
-    "float gl_ClipDistance[6];\n"
+    "out float gl_ClipDistance[6];\n"
```

That alone would let the picking shader compile on any GLSL 4.00+ driver. The `#version 410` issue is a separate compatibility decision (some drivers cap at 4.00) and would warrant a CMake feature check or a `#version 400` baseline.
